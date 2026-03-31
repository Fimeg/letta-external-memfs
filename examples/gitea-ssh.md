# Gitea with SSH Key Auth

## Setup

1. Generate an SSH key (if you don't have one):
   ```bash
   ssh-keygen -t ed25519 -C "letta@example.com"
   ```

2. Add the public key to Gitea (Settings → SSH / GPG Keys → Add Key)

3. Ensure the key is loaded in ssh-agent:
   ```bash
   eval "$(ssh-agent -s)"
   ssh-add ~/.ssh/id_ed25519
   ```

## Environment Variable

```bash
export LETTA_MEMFS_GIT_URL="git@gitea.example.com:username/repo.git"
```

Or for a custom port:
```bash
export LETTA_MEMFS_GIT_URL="ssh://git@gitea.example.com:2222/username/repo.git"
```

## Per-Agent Repos

```bash
export LETTA_MEMFS_GIT_URL="git@gitea.example.com:username/{agentId}.git"
```

## SSH Config (Recommended)

Add to `~/.ssh/config`:

```
Host gitea-letta
    HostName gitea.example.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

Then use:
```bash
export LETTA_MEMFS_GIT_URL="git@gitea-letta:username/repo.git"
```

## Initial Repo Setup

```bash
cd ~/.letta/agents/{agentId}/memory
git init
git remote add origin git@gitea.example.com:username/repo.git
git checkout -b daemon
git add . && git commit -m "init"
git push -u origin daemon
```

## Troubleshooting

**"Permission denied (publickey)":**
- Ensure key is in ssh-agent: `ssh-add -l`
- Test with: `ssh -T git@gitea.example.com`
- Check Gitea has the public key

**"Could not resolve hostname":**
- Verify SSH config HostName matches your network
- For local testing, use IP address: `git@10.10.20.120:user/repo.git`

**Passphrase prompts:**
The agent process runs non-interactively and can't enter passphrases. Either:
- Use a key without passphrase
- Ensure ssh-agent has the key loaded before starting the bridge
- Use `ssh-add` in your shell startup
