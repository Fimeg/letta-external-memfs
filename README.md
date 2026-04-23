# Letta External MemFS

Route Letta's git-backed memory (MemFS) through an external git host (Gitea, GitHub, GitLab) instead of Letta's built-in `/v1/git/` proxy. No sidecar required.

> **Status:** Endorsed by Letta team as the recommended self-hosted memfs path. Community-maintained, not part of Letta support SLA.

## What This Does

By default, Letta's git-backed memory (MemFS) expects clients to push to the server's own `/v1/git/` proxy, which writes to GCS in Cloud or a local bare repo when self-hosted. This repo provides two patches that together retarget that flow at an external git host:

- **`patches/memoryGit.ts.patch`** (client, letta-code) — routes the client's git push/pull to `LETTA_MEMFS_GIT_URL` instead of the server proxy.
- **`patches/server_memory_sync_endpoint.patch`** (server, letta-server) — exposes `POST /v1/agents/{agent_id}/memory/sync-from-git` so the server can re-read files into its postgres block cache after an external pull. This replaces the files→blocks materialization that normally fires on `/v1/git/` receive and isn't triggered by the client patch alone.

## How It Works

```
Without these patches:
  letta-code → Letta server (/v1/git/ proxy) → bare repo → postgres cache sync

With these patches:
  letta-code → external git host (Gitea/GitHub/GitLab)
                                 ↓
              sync mechanism pulls on server side
                                 ↓
              POST /v1/agents/{id}/memory/sync-from-git → postgres cache
```

### Alignment with Letta's model

Letta's own `block_manager_git.py` treats git as the source of truth and postgres as a cache. The external git host plays the role Letta's internal bare repo plays in stock: **canonical, durable, the thing clients and servers both resolve against**. The server's local memfs is a working checkout of it. Postgres blocks are a cache rebuilt from files.

This framing keeps the single-machine case simple (server and client share a filesystem; the external remote is backup) and makes the multi-machine case coherent (any machine can rebuild from the external remote).

## Requirements

- Letta server 0.16.6+ (self-hosted, patched)
- Letta Code 0.19.6+ (patched)
- External git host (tested on Gitea, GitHub, GitLab)
- Git token with repo read/write access
- Docker + ability to rebuild your Letta server image

## Installation

### 1. Patch Letta Code (client)

```bash
cd /path/to/letta-code
patch -p1 < /path/to/letta-external-memfs/patches/memoryGit.ts.patch
```

Three small additive hunks in `src/agent/memoryGit.ts` — no existing logic is removed.

### 2. Build Letta Code

```bash
cd /path/to/letta-code
bun run build
```

### 3. Patch Letta Server

```bash
cd /path/to/letta-source      # your letta-server checkout
patch -p1 < /path/to/letta-external-memfs/patches/server_memory_sync_endpoint.patch
```

Adds a single new route (`POST /v1/agents/{agent_id}/memory/sync-from-git`) and one import line. No existing code is modified. See [`patches/README.md`](patches/README.md) for details on what it does and why it's needed.

### 4. Rebuild Letta Server Image

Your build command, versioned so you can roll back if needed:

```bash
cd /path/to/letta-source
docker build -t letta-local:<version>-patched-vN .
```

Then update your `docker-compose.yml` to reference the new tag and `docker compose up -d`.

### 5. Configure Server

In `docker-compose.yml` (or whatever compose file you use), set memfs to local backing so the server's `GitEnabledBlockManager` writes to the container filesystem:

```yaml
services:
  letta-server:
    image: letta-local:<version>-patched-vN
    environment:
      - LETTA_MEMFS_SERVICE_URL=local
    volumes:
      # Persistence: server memfs survives container recreation
      - letta-memfs:/root/.letta/memfs

volumes:
  letta-memfs:
```

The volume matters. Without it, `docker compose down` wipes the server's local memfs and session init fails on restart (the agent's postgres blocks survive, but the files are gone). See [`docker-compose.override.yml`](docker-compose.override.yml) for a ready-to-apply snippet.

Or set in `conf.yaml`:
```yaml
letta:
  memfs_service_url: "local"
```

### 6. Configure Client

Add to your bridge service or shell environment:

```bash
export LETTA_MEMFS_GIT_URL="https://token@gitea.example.com/user/repo.git"
```

Or with `{agentId}` placeholder for per-agent repos:
```bash
export LETTA_MEMFS_GIT_URL="https://token@gitea.example.com/agents/{agentId}.git"
```

#### Local Instance Optimization

If your git host is on the same network as your agent (e.g., internal IP like `10.10.20.120`), enable client-side short-circuit:

```bash
export LETTA_MEMFS_LOCAL=1
```

This avoids unnecessary network hops for local git operations.

### 7. Prepare Your Git Repo

Create a repo on your git host, then initialize the memory directory:

```bash
cd ~/.letta/agents/{agentId}/memory
git init
git remote add origin https://token@gitea.example.com/user/repo.git
git checkout -b main
```

### 8. Enable MemFS for Your Agent

Agent-level enablement is separate from the infrastructure env vars. From a Letta Code CLI session against your server:

```
/memfs enable
```

Or via API:

```bash
curl -X PATCH "http://localhost:8283/v1/agents/<agent_id>" \
  -H "Authorization: Bearer $LETTA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"tags": ["git-memory-enabled"]}'
```

The server tags the agent with `git-memory-enabled`, creates the agent's git repo on its backing memfs, and commits existing blocks as the initial state. After this, `GitEnabledBlockManager` handles all block operations with git as source of truth, postgres as cache.

## Configuration Reference

| Variable | Purpose | Example |
|----------|---------|---------|
| `LETTA_MEMFS_SERVICE_URL` | Server-side: use local memfs backing | `local` |
| `LETTA_MEMFS_GIT_URL` | Client-side: external git URL | `https://token@host/user/repo.git` |
| `LETTA_MEMFS_LOCAL` | Client-side: short-circuit for same-network hosts | `1` |

### New API route (added by the server patch)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/agents/{agent_id}/memory/sync-from-git` | Rebuild postgres block cache from the server's backing git repo. Optional `?recompile=true` chains a system-prompt rebuild. |

## Server-Side Sync

The client patch routes `letta-code` to your external git host. That push never reaches the server's `/v1/git/` receive, so the files→postgres cache materialization that normally fires on receive doesn't run. Two things must happen on the server to close the loop:

1. **Pull** the server's backing memfs from your external git host
2. **Sync** the postgres block cache from those pulled files (the new endpoint)

### Minimal sync pattern

```bash
AGENT_ID=<your-agent-id>
BRANCH=main
CONTAINER=<your-letta-container>

# 1. Pull external remote into server's local memfs
docker exec "$CONTAINER" bash -c "
  cd /root/.letta/memfs/$AGENT_ID &&
  git fetch origin &&
  git reset --hard origin/$BRANCH
"

# 2. Tell the server to rebuild its postgres cache from the pulled files
curl -X POST "http://localhost:8283/v1/agents/$AGENT_ID/memory/sync-from-git?recompile=false" \
  -H "Authorization: Bearer $LETTA_API_KEY"
```

### When to call `?recompile=true`

`sync-from-git` alone updates the postgres block cache but doesn't rebuild the system prompt. Since recompilation invalidates the KV cache on the next inference call (see [Cameron's guidance on recompile cost](#recompile-cost)), prefer to let recompile happen naturally on the next compaction unless you actively need the prompt to reflect new content *right now*.

A diff-gated pattern saves tokens on long-running agents:

```bash
# Only recompile if system/ actually changed
BEFORE=$(docker exec "$CONTAINER" bash -c "cd /root/.letta/memfs/$AGENT_ID && git rev-parse HEAD -- system/ 2>/dev/null")
docker exec "$CONTAINER" bash -c "cd /root/.letta/memfs/$AGENT_ID && git fetch origin && git reset --hard origin/$BRANCH"
AFTER=$(docker exec "$CONTAINER" bash -c "cd /root/.letta/memfs/$AGENT_ID && git rev-parse HEAD -- system/ 2>/dev/null")

if [ "$BEFORE" != "$AFTER" ]; then
  curl -X POST "http://localhost:8283/v1/agents/$AGENT_ID/memory/sync-from-git?recompile=true" -H "Authorization: Bearer $LETTA_API_KEY"
else
  curl -X POST "http://localhost:8283/v1/agents/$AGENT_ID/memory/sync-from-git?recompile=false" -H "Authorization: Bearer $LETTA_API_KEY"
fi
```

### <a name="recompile-cost"></a>Recompile cost

Letta's maintainers have noted that `rebuild_system_prompt` is **expensive in token terms** — it invalidates the LLM provider's KV cache for the system prompt, so the next inference call reprocesses the full prompt. Sync often (cheap); recompile only when `system/` actually changed (not cheap). The default `?recompile=false` reflects this.

### Sync cadence

Pick based on how actively the agent is edited:
- **Cron** (15–30 min): simple, tolerant of transient Gitea downtime
- **Event-driven** (Gitea webhook → server host): lower latency if you need it
- **Both**: webhook for latency, cron as belt-and-suspenders

### Single-machine deployments

If your server and client share a filesystem (both on one host, client writes directly to the path the server reads), you don't need the pull step — the file change is immediately visible to the server. You still need step 2 (the sync endpoint) because postgres is a separate state layer and the client's push bypassed the server's receive hook that normally updates it.

## URL Formats

### Gitea / GitLab / GitHub (token in URL)
```bash
https://token@gitea.example.com/username/repo.git
```

### With agentId placeholder (per-agent repos)
```bash
https://token@gitea.example.com/agents/{agentId}.git
```

### SSH (key-based auth)
```bash
git@gitea.example.com:user/repo.git
```

For SSH, ensure your key is available to the agent process (e.g., ssh-agent, or key in `~/.ssh/` with proper permissions). The URL in `LETTA_MEMFS_GIT_URL` should use the SSH format, and the key must be passphrase-less or loaded in ssh-agent.

### ⚠️ Security Note: Tokens in URLs

When using `https://token@host/...` format, be aware that:
- The token appears in `ps aux` output while git operations run
- The URL with token may be stored in git config (`git remote get-url origin`)
- Shell history may record the export command

**Recommendations:**
- Use `git config credential.helper store` to cache credentials separately
- Or use SSH key auth (`git@host:user/repo.git`) instead of HTTPS tokens
- For production, consider a git credential helper that reads from environment or vault

## Testing

After starting your bridge, verify:

```bash
# Check remote URL
cd ~/.letta/agents/{agentId}/memory
git remote -v

# Test push
echo "# test" >> system/test.md
git add . && git commit -m "test"
git push origin main
```

## Files in This Repo

| File | Purpose |
|------|---------|
| `patches/memoryGit.ts.patch` | Patch for letta-code (client) |
| `patches/server_memory_sync_endpoint.patch` | Patch for letta-server (adds `/memory/sync-from-git`) |
| `patches/README.md` | Patch application order + tested Letta versions |
| `docker-compose.override.yml` | Example override with persistence volume |
| `systemd/bridge.service.example` | Example systemd service for the client bridge |
| `examples/` | URL configs for various git hosts |

## Troubleshooting

### "fatal: unable to access... SSL certificate problem"
Your git host uses a self-signed cert. Either:
- Add the cert to system trust store
- `git config --global http.sslVerify false` (insecure, dev only)

### "remote: 401 Unauthorized"
Token is wrong or missing. Verify token in URL:
```bash
git remote get-url origin
```

### "could not read transcript"
Normal on first run — the transcript file is created after first message.

### `<projection>` paths point to files that don't exist
The agent's system prompt is compiled from postgres blocks. If the files on disk are newer than the blocks (a common drift in multi-machine setups), the compiled prompt references stale paths. Sync the server's cache:

```bash
curl -X POST "http://localhost:8283/v1/agents/<agent_id>/memory/sync-from-git?recompile=true" \
  -H "Authorization: Bearer $LETTA_API_KEY"
```

If that alone doesn't resolve it, the server's backing memfs is probably behind the external remote — pull it first (see [Server-Side Sync](#server-side-sync)).

### `fatal: refusing to merge unrelated histories`
The server's backing memfs was initialized independently of your external remote. Inside the container:

```bash
cd /root/.letta/memfs/<agent_id>
git fetch origin
git reset --hard origin/<branch>
```

This aligns the server's checkout to the remote. Safe because postgres is the cache, not the canonical state — the following `sync-from-git` call will rebuild it from the reset files.

### Session init fails: "Memfs not enabled"
Agent-level enablement is separate from infrastructure env vars. Run `/memfs enable` in a Letta Code session against the server, or `PATCH /v1/agents/<id>` with the `git-memory-enabled` tag. See [Step 8](#8-enable-memfs-for-your-agent).

## Status

- ✅ Tested: Gitea, GitHub, GitLab with HTTPS token auth, `main` branch
- 🔄 Should work: Per-agent repos with `{agentId}` placeholder
- ❓ Not tested: SSH key auth

## License

MIT — same as Letta Code.

## Credit

Developed for self-hosted Letta deployments. External MemFS eliminates the need for a sidecar proxy — direct git connections, centralized storage, full web UI for memory management.
