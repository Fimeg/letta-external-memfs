# Letta External MemFS

Connect Letta Code directly to external git hosts (Gitea, GitLab, GitHub, etc.) for memory persistence — no memfs sidecar required.

> **Status:** Endorsed by Letta team as the recommended self-hosted memfs path. Community-maintained, not part of Letta support SLA.

## What This Does

By default, Letta's git-backed memory (MemFS) requires either:
- Letta Cloud (managed)
- Letta server OSS with local storage (works, but git transport is via `/v1/git/` proxy)

Some users added a **sidecar proxy** (`git http-backend` wrapper) to handle git transport locally.

This patch eliminates the sidecar and **connects directly** to any git host that supports HTTPS + token auth, or SSH + key auth.

## How It Works

```
Without this patch:
  letta-code → Letta server → /v1/git/ proxy → sidecar → bare repo

With this patch:
  letta-code → Gitea/GitLab/GitHub (direct HTTPS)
```

The Letta server still uses local memfs for block operations, but git transport bypasses the proxy entirely.

## Requirements

- Letta server 0.16.6+ (self-hosted)
- Letta Code 0.19.6+
- External git host (tested on Gitea, should work with GitLab/GitHub)
- Git token with repo read/write access

## Installation

### 1. Patch Letta Code

Apply the patch to `src/agent/memoryGit.ts`:

```bash
cd /path/to/letta-code
patch -p1 < /path/to/letta-external-memfs/patches/memoryGit.ts.patch
```

Or manually edit — see the patch file for the exact changes (3 small additions).

### 2. Build Letta Code

```bash
cd /path/to/letta-code
bun run build
```

### 3. Configure Server

Edit your `docker-compose.yml`:

```yaml
services:
  letta-server:
    environment:
      # Use local memfs for block operations (not sidecar)
      - LETTA_MEMFS_SERVICE_URL=local
```

Or set in `conf.yaml`:
```yaml
letta:
  memfs_service_url: "local"
```

### 4. Configure Client

Add to your bridge service or shell environment:

```bash
export LETTA_MEMFS_GIT_URL="https://token@gitea.example.com/user/repo.git"
```

Or with `{agentId}` placeholder for per-agent repos:
```bash
export LETTA_MEMFS_GIT_URL="https://token@gitea.example.com/agents/{agentId}.git"
```

#### Local Instance Optimization

If your Gitea instance is on the same network as your agent (e.g., internal IP like `10.10.20.120`), enable client-side short-circuit:

```bash
export LETTA_MEMFS_LOCAL=1
```

This avoids unnecessary network hops for local git operations.

### 5. Prepare Your Git Repo

Create a repo on your git host, then initialize the memory directory:

```bash
cd ~/.letta/agents/{agentId}/memory
git init
git remote add origin https://token@gitea.example.com/user/repo.git
git checkout -b main    # Use 'daemon' branch for high-frequency writes (see Advanced Patterns)
```

## Configuration Reference

| Variable | Purpose | Example |
|----------|---------|---------|
| `LETTA_MEMFS_SERVICE_URL` | Server-side: use local memfs | `local` |
| `LETTA_MEMFS_GIT_URL` | Client-side: external git URL | `https://token@host/user/repo.git` |

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
git push origin daemon
```

## Files in This Repo

| File | Purpose |
|------|---------|
| `patches/memoryGit.ts.patch` | Patch for letta-code |
| `docker-compose.override.yml` | Example Docker Compose config |
| `systemd/bridge.service.example` | Example systemd service |
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

## Status

- ✅ Tested: Gitea with HTTP token auth
- ✅ Tested: Single repo, multiple branches (daemon, intrusive, main)
- 🔄 Should work: GitLab, GitHub (token auth)
- 🔄 Should work: Per-agent repos with `{agentId}` placeholder
- ❓ Not tested: SSH key auth

## License

MIT — same as Letta Code.

## Credit

Developed for self-hosted Letta deployments. External MemFS eliminates the need for a sidecar proxy — direct git connections, centralized storage, full web UI for memory management.
