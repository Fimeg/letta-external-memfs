# Issue Template

Thank you for reporting an issue with letta-external-memfs. Please provide the following information:

## Environment

- **Letta Code version:** (e.g., 0.22.4)
- **Letta Server version:** (e.g., 0.16.6)
- **Git host:** (e.g., Gitea 1.23, GitLab 17.0, GitHub)
- **Authentication method:** (HTTPS token / SSH key)
- **Operating system:** (e.g., Ubuntu 24.04, Fedora 42)

## Issue Description

What happened? What did you expect to happen?

## Steps to Reproduce

1. 
2. 
3. 

## Configuration

Please include (with sensitive info redacted):
- `LETTA_MEMFS_GIT_URL` format (e.g., `https://token@gitea.example.com/user/repo.git`)
- Server `docker-compose.yml` or `conf.yaml` memfs settings
- Any relevant environment variables

## Logs

Include relevant log output:
```
# From letta-code bridge
# From letta-server
# From git operations (if applicable)
```

## Checklist

- [ ] I have verified my token/key has proper permissions
- [ ] I have tested the git URL manually (`git clone` or `git push`)
- [ ] I have checked the [Troubleshooting section](../README.md#troubleshooting)
- [ ] I am using the latest version of letta-external-memfs
