# Changelog

## [1.0.0] - 2026-04-16

### Added
- Initial release: External MemFS for Letta
- Direct git host connection (Gitea, GitLab, GitHub) without sidecar
- HTTPS token auth and SSH key auth support
- Per-agent repository support with `{agentId}` placeholder
- Local instance optimization (`LETTA_MEMFS_LOCAL=1`)
- E2EE architecture with session persistence

### Documentation
- Setup guides for Gitea (HTTPS and SSH)
- Docker Compose and systemd examples
- Security notes on token handling
- Troubleshooting guide

## [1.0.1] - 2026-04-16

### Changed
- Changed default branch from `daemon` to `main` in examples
- Added endorsement note: "Endorsed by Letta team as the recommended self-hosted memfs path"
- Added security warning about tokens-in-URL
- Documented `LETTA_MEMFS_LOCAL=1` optimization

### Added
- CHANGELOG.md
- GitHub issue template
