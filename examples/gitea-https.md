# Gitea with HTTPS Token Auth

## Setup

1. Create a personal access token in Gitea (Settings → Applications → Generate Token)
2. Grant the token `repo` scope (read/write access)

## Environment Variable

```bash
export LETTA_MEMFS_GIT_URL="https://your-token@gitea.example.com/username/repo.git"
```

## Per-Agent Repos

If you want separate repos for each agent:

```bash
export LETTA_MEMFS_GIT_URL="https://your-token@gitea.example.com/username/{agentId}.git"
```

The `{agentId}` placeholder will be replaced with the actual agent ID at runtime.

## Initial Repo Setup

```bash
# On your git host, create the repo
# Then locally:
cd ~/.letta/agents/{agentId}/memory
git init
git remote add origin https://your-token@gitea.example.com/username/repo.git
git checkout -b main
# Create initial commit
git add . && git commit -m "init"
git push -u origin main
```

### Advanced: High-Frequency Writes (Optional)

If you have agents writing frequently (e.g., every heartbeat), use a `daemon` branch pattern:

```bash
git checkout -b daemon
git push -u origin daemon
```

## Troubleshooting

**Self-signed certificates:**
If Gitea uses a self-signed cert, git will fail. Either:
- Add the cert to system trust store
- `git config --global http.sslVerify false` (dev only)

**401 Unauthorized:**
- Verify token has `repo` scope
- Check token hasn't expired
- Ensure token is URL-encoded if it contains special characters
