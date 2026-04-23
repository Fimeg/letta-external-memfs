# Changelog

## [Unreleased]

### Added
- **Server patch:** `patches/server_memory_sync_endpoint.patch` exposes `POST /v1/agents/{agent_id}/memory/sync-from-git`, wrapping Letta's existing `GitEnabledBlockManager.sync_blocks_from_git` over HTTP. Optional `?recompile=true` chains a system prompt rebuild. Tested against letta-server commit `a1333365` (0.16.7).
- `patches/README.md` — documents both patches, their purpose, apply order, and tested versions.
- README section **Server-Side Sync** with a minimal sync pattern and a diff-gated recompile pattern.
- README section **Enabling MemFS for Your Agent** covering `/memfs enable` and the `git-memory-enabled` tag.
- README architecture note aligning the patch's model with Letta's own "git is source of truth, postgres is cache" framing (`letta/services/block_manager_git.py`).
- `docker-compose.override.yml`: named volume on `/root/.letta/memfs` so server state survives `docker compose down`.
- Troubleshooting entries for stale `<projection>` paths, `refusing to merge unrelated histories`, and "Memfs not enabled" on session init.

### Changed
- Installation is now 8 steps covering both client and server patches (was 5, client-only).
- Configuration Reference adds `LETTA_MEMFS_LOCAL` and the new API route.

### Removed
- `daemon` branch pattern from all examples and docs — `main` is now the sole documented branch

### Fixed
- README testing section pushed to `daemon`; now pushes to `main` consistent with setup
- Patch docstring example used plaintext `http://`; now `https://` in line with security guidance
- CHANGELOG 1.0.0 entry mentioned "E2EE architecture with session persistence" — stray content from unrelated work, removed

## [1.0.0] - 2026-04-16

### Added
- Initial release: External MemFS for Letta
- Direct git host connection (Gitea, GitLab, GitHub) without sidecar
- HTTPS token auth and SSH key auth support
- Per-agent repository support with `{agentId}` placeholder
- Local instance optimization (`LETTA_MEMFS_LOCAL=1`)

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
