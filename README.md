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

# Required — adds the sync-from-git endpoint
patch -p1 < /path/to/letta-external-memfs/patches/server_memory_sync_endpoint.patch

# Opt-in hardening (recommended for multi-machine users)
patch -p1 < /path/to/letta-external-memfs/patches/server_sync_delete_propagation.patch
patch -p1 < /path/to/letta-external-memfs/patches/server_system_only_blocks.patch
```

- `server_memory_sync_endpoint.patch` — adds `POST /v1/agents/{agent_id}/memory/sync-from-git`.
- `server_sync_delete_propagation.patch` — makes `sync_blocks_from_git` also remove postgres blocks whose files were deleted in git (otherwise orphan blocks accumulate forever).
- `server_system_only_blocks.patch` — gates path → block mapping behind `LETTA_MEMFS_BLOCK_PATH_PREFIXES` so you can restrict blocks to `system/` (or any prefix set) and use the rest of the repo as filesystem-only backup.

The last two are independent and opt-in. See [`patches/README.md`](patches/README.md) for details.

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

Create a repo on your git host, then initialize the memory directory. The setup differs depending on whether the remote is empty or already has content — get this wrong and you'll hit `refusing to merge unrelated histories` on the first push:

```bash
cd ~/.letta/agents/{agentId}/memory
git init
git remote add origin https://token@gitea.example.com/user/repo.git

# Fetch first to see what's there
git fetch origin 2>/dev/null

# If remote is empty (fetch returned nothing): initialize fresh
if ! git rev-parse --verify origin/main >/dev/null 2>&1; then
  git checkout -b main
# If remote already has content (e.g., you pushed from another machine first):
else
  git checkout -b main --track origin/main
fi
```

Getting this wrong at setup is the most common cause of the `fatal: refusing to merge unrelated histories` error documented in [Troubleshooting](#troubleshooting) — the fetch-first pattern avoids it entirely.

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
| `LETTA_MEMFS_BLOCK_PATH_PREFIXES` | Server-side (with `server_system_only_blocks.patch`): comma-separated path prefixes that are allowed to become blocks. Unset = upstream behavior (every `.md` becomes a block). `skills/**/SKILL.md` is always allowed. | `system/` |

### New API route (added by the server patch)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/agents/{agent_id}/memory/sync-from-git` | Rebuild postgres block cache from the server's backing git repo. Optional `?recompile=true` chains a system-prompt rebuild. |

## Server-Side Sync

The client patch routes `letta-code` to your external git host. That push never reaches the server's `/v1/git/` receive, so the files→postgres cache materialization that normally fires on receive doesn't run. Two things must happen on the server to close the loop:

1. **Pull** the server's backing git storage from your external git host
2. **Sync** the postgres block cache from those pulled objects (the new endpoint)

> **Single-machine deployments first:** if your server and client share a filesystem (e.g., one host, client writing to the same volume the server reads) you can skip step 1 — the file change is already visible to the server. You still need step 2 because postgres is a separate state layer that the client's push bypassed. Most self-hosted users land in this bucket; the multi-machine pattern below is only needed when the server and client are on different hosts.

### Where the server actually stores memfs

Letta's `LocalStorageBackend` stores each agent's git objects as a **bare repo** at:

```
~/.letta/memfs/repository/{org_id}/{agent_id}/repo.git/
```

Inside the container (default install): `/root/.letta/memfs/repository/{org_id}/{agent_id}/repo.git/`.

This is the path `sync_blocks_from_git` reads from. There is **no working checkout** at `/root/.letta/memfs/{agent_id}/` on the server — that path does not exist. Any sync pattern that targets a working checkout is a no-op. Fetches must target the bare repo with `--git-dir` (and `--update-head-ok`, since the bare repo's `HEAD` points at the branch you're updating).

For default self-hosted installs, `org_id` is `org-00000000-0000-4000-8000-000000000000` ([`letta/constants.py` `DEFAULT_ORG_ID`](https://github.com/letta-ai/letta/blob/0.16.7/letta/constants.py)). If you're on a custom org, discover it from the filesystem (there's typically one directory):

```bash
ORG_ID=$(docker exec "$CONTAINER" ls /root/.letta/memfs/repository | head -1)
```

> **Credit:** the path/architecture finding was reported by [@PurposeUnknown](https://github.com/PurposeUnknown) after testing an earlier version of these docs; the old pattern targeted `/root/.letta/memfs/$AGENT_ID` (a working-checkout path that doesn't exist on the server).

### Minimal sync pattern (multi-machine)

```bash
AGENT_ID=<your-agent-id>
BRANCH=main
CONTAINER=<your-letta-container>
GIT_URL=<your-external-git-url-including-token>
ORG_ID=${ORG_ID:-org-00000000-0000-4000-8000-000000000000}

REPO_DIR="/root/.letta/memfs/repository/$ORG_ID/$AGENT_ID/repo.git"

# 1. Fetch from the external remote directly into the bare repo.
#    --update-head-ok is required because HEAD in the bare repo points at
#    the branch being updated. Fetching by refspec (main:main) updates the
#    local ref in place without a working checkout (which doesn't exist).
docker exec "$CONTAINER" git --git-dir="$REPO_DIR" \
  fetch --force --update-head-ok "$GIT_URL" "$BRANCH:$BRANCH"

# 2. Tell the server to rebuild its postgres cache from the fetched objects.
curl -X POST "http://localhost:8283/v1/agents/$AGENT_ID/memory/sync-from-git?recompile=false" \
  -H "Authorization: Bearer $LETTA_API_KEY"
```

If you'd rather persist the remote in the bare repo's config (so subsequent fetches can omit `$GIT_URL`), run once:

```bash
docker exec "$CONTAINER" git --git-dir="$REPO_DIR" remote add origin "$GIT_URL"
# Subsequent fetches:
docker exec "$CONTAINER" git --git-dir="$REPO_DIR" fetch --force --update-head-ok origin "$BRANCH:$BRANCH"
```

### When to call `?recompile=true`

`sync-from-git` alone updates the postgres block cache but doesn't rebuild the system prompt. Since recompilation invalidates the KV cache on the next inference call (see [recompile cost](#recompile-cost)), prefer to let recompile happen naturally on the next compaction unless you actively need the prompt to reflect new content *right now*.

A diff-gated pattern saves tokens on long-running agents — only recompile when `system/` actually changed between the old and new tip of `$BRANCH`:

```bash
REPO_DIR="/root/.letta/memfs/repository/$ORG_ID/$AGENT_ID/repo.git"

BEFORE=$(docker exec "$CONTAINER" git --git-dir="$REPO_DIR" rev-parse "$BRANCH^{tree}:system" 2>/dev/null || echo missing)
docker exec "$CONTAINER" git --git-dir="$REPO_DIR" \
  fetch --force --update-head-ok "$GIT_URL" "$BRANCH:$BRANCH"
AFTER=$(docker exec "$CONTAINER" git --git-dir="$REPO_DIR" rev-parse "$BRANCH^{tree}:system" 2>/dev/null || echo missing)

if [ "$BEFORE" != "$AFTER" ]; then
  curl -X POST "http://localhost:8283/v1/agents/$AGENT_ID/memory/sync-from-git?recompile=true"  -H "Authorization: Bearer $LETTA_API_KEY"
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
| `patches/server_memory_sync_endpoint.patch` | Server patch — adds `POST /v1/agents/{id}/memory/sync-from-git` (required) |
| `patches/server_sync_delete_propagation.patch` | Server patch — removes orphan postgres blocks when files are deleted in git (opt-in) |
| `patches/server_system_only_blocks.patch` | Server patch — env-gated path-prefix filter for block mapping (opt-in) |
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
The agent's system prompt is compiled from postgres blocks. If the files in the server's bare repo are newer than the blocks (a common drift in multi-machine setups), the compiled prompt references stale paths. Sync the server's cache:

```bash
curl -X POST "http://localhost:8283/v1/agents/<agent_id>/memory/sync-from-git?recompile=true" \
  -H "Authorization: Bearer $LETTA_API_KEY"
```

If that alone doesn't resolve it, the server's backing bare repo is probably behind the external remote — fetch into it first (see [Server-Side Sync](#server-side-sync)).

### `fatal: refusing to merge unrelated histories`
The server's backing bare repo was initialized independently of your external remote. Fix by fetching into the bare repo with a refspec (no merge involved):

```bash
ORG_ID=${ORG_ID:-org-00000000-0000-4000-8000-000000000000}
REPO_DIR=/root/.letta/memfs/repository/$ORG_ID/<agent_id>/repo.git
docker exec "$CONTAINER" git --git-dir="$REPO_DIR" \
  fetch --force --update-head-ok "$GIT_URL" "<branch>:<branch>"
```

`--force` lets the fetch overwrite the server's divergent branch with the remote's canonical tip. Safe because postgres is the cache, not the canonical state — the following `sync-from-git` call will rebuild it from the fetched objects.

### Sync endpoint returns 500 and logs `UnicodeDecodeError`
Letta's git read path (`letta/services/memory_repo/git_operations.py`) runs `subprocess.run(..., text=True)`, which tries to UTF-8 decode every file in the repo. Any binary file (PDFs, images, macOS `._*` resource forks, `.DS_Store`) crashes the sync.

Workaround: keep the memfs repo markdown-only. Add a `.gitignore` at the repo root:

```gitignore
# macOS
.DS_Store
._*

# Binaries — remove if you need them for another workflow
*.pdf
*.png
*.jpg
*.jpeg
*.gif
*.webp
*.zip
*.tar.gz
```

### Deleted files still show up as blocks
`GitEnabledBlockManager.sync_blocks_from_git` (`letta/services/block_manager_git.py`) rebuilds the postgres cache by iterating the git blocks and upserting each — it does not scan postgres for blocks whose source files no longer exist in git. Deleting or renaming a file in the repo leaves the old postgres block behind as an orphan.

Workaround: delete orphaned blocks via the API:

```bash
curl -X DELETE "http://localhost:8283/v1/agents/<agent_id>/core-memory/blocks/<block_label>" \
  -H "Authorization: Bearer $LETTA_API_KEY"
```

### Non-`system/` files unexpectedly appear as memory blocks
`letta/services/memory_repo/path_mapping.py::memory_block_label_from_markdown_path` maps *every* `.md` file under the agent repo to a postgres block label, not just files under `system/` (the one exception is `skills/**/SKILL.md`, which has its own special case). Files under `reference/`, `users/`, etc. all become blocks — pinning them into the compiled system prompt is a separate agent-level concern, but at the block level they exist.

If you want repo-as-backup with only `system/` as blocks, keep non-block content out of the agent's memfs repo entirely until/unless this behavior changes upstream.

### Session init fails: "Memfs not enabled"
Agent-level enablement is separate from infrastructure env vars. Run `/memfs enable` in a Letta Code session against the server, or `PATCH /v1/agents/<id>` with the `git-memory-enabled` tag. See [Step 8](#8-enable-memfs-for-your-agent).

## Status

- ✅ Tested: Gitea, GitHub, GitLab with HTTPS token auth, `main` branch
- ✅ Tested: persistence volume on named volume and bind mount
- ✅ Tested: multi-machine sync via `--git-dir` bare-repo fetch (add/remove file on machine A → sync-from-git on machine B reflects change)
- 🔄 Should work: Per-agent repos with `{agentId}` placeholder
- ❓ Not tested: SSH key auth

## Known upstream Letta behaviors that affect users of this patch set

These are existing behaviors in `letta-server` that surface when running `GitEnabledBlockManager`, not regressions from this patch set. The workarounds below are scoped to what users of this repo can do today without upstream changes.

| Behavior | Source | User-facing effect | This repo's answer |
|---|---|---|---|
| `sync_blocks_from_git` is additive-only | `letta/services/block_manager_git.py` | File deletions/renames in git leave orphan postgres blocks | **Fixed by opt-in patch** `patches/server_sync_delete_propagation.patch` — sync now also removes orphan blocks. |
| `subprocess.run(..., text=True)` in git read path | `letta/services/memory_repo/git_operations.py` | Binary files in the repo cause `UnicodeDecodeError` → HTTP 500 from sync endpoint | Documented workaround only: keep the repo markdown-only via `.gitignore` (see [Troubleshooting](#sync-endpoint-returns-500-and-logs-unicodedecodeerror)). No patch shipped since the repo is intended to be markdown-only. |
| `path_mapping` maps all `.md` paths to block labels | `letta/services/memory_repo/path_mapping.py` | Files under `reference/`, `users/`, etc. become blocks alongside `system/` files | **Fixed by opt-in patch** `patches/server_system_only_blocks.patch` — set `LETTA_MEMFS_BLOCK_PATH_PREFIXES=system/` to restrict which paths become blocks. |

The patches default-off the hardening (unset env var = upstream behavior unchanged) so applying them doesn't change behavior for anyone who hasn't deliberately opted in. Report in Letta's Discord if any become painful for your deployment; the Letta team prefers chat triage over GitHub issues for questions like these.

## License

MIT — same as Letta Code.

## Credit

Developed for self-hosted Letta deployments. External MemFS eliminates the need for a sidecar proxy — direct git connections, centralized storage, full web UI for memory management.

### Acknowledgements

- **[@PurposeUnknown](https://github.com/PurposeUnknown)** — multi-machine testing that identified the bare-repo storage path, the `org_id` prefix in the storage layout, and the upstream Letta behaviors documented in the table above. The `--git-dir` sync pattern in this README exists because of this work.
- **Letta team (Ezra, Cameron, Lucia)** — review feedback on the upstream PR draft and recompile-cost guidance.
