# Changelog

## [Unreleased]

### Fixed
- **Critical shared-block deletion bug in `server_sync_delete_propagation.patch`.** The `_delete_block_from_postgres` method was deleting blocks from ALL agents when called for one agent, breaking multi-agent setups with shared blocks. Fixed by scoping the `BlocksAgents` delete to `agent_id` and only hard-deleting blocks that are globally orphaned. Reported by Zandoodlee (@yankzan) on 2026-04-25. Affects all multi-agent deployments using memfs with shared blocks. See [`server_fix_shared_block_deletion.patch`](patches/server_fix_shared_block_deletion.patch).
- **Documented server sync path was wrong.** The previous `docker exec cd /root/.letta/memfs/$AGENT_ID && git fetch && git reset --hard` pattern targeted a working-checkout path that does not exist on the server. Letta's `LocalStorageBackend` stores bare repo objects at `~/.letta/memfs/repository/{org_id}/{agent_id}/repo.git/`, and that is where `sync_blocks_from_git` reads from. Replaced with `git --git-dir=<bare-repo-path> fetch --force --update-head-ok <url> <branch>:<branch>`. Discovered by [@PurposeUnknown](https://github.com/PurposeUnknown) during multi-machine testing.
- **`org_id` is part of the server storage path.** Default self-hosted install uses `org-00000000-0000-4000-8000-000000000000`. README now documents discovery (`docker exec ... ls .../repository | head -1`) and parameterizes `$ORG_ID` in every sync snippet.
- **Step 7 divergent-history footgun.** `git checkout -b main` after `git remote add` collided with a populated remote and produced `fatal: refusing to merge unrelated histories` on first push. Now fetches first and conditionally creates tracking branch vs. fresh branch.
- **Troubleshooting `refusing to merge unrelated histories`** rewritten to use `--git-dir` fetch with refspec (no merge involved) instead of `git reset --hard` in a non-existent working checkout.

### Added
- **Server patch:** `patches/server_fix_shared_block_deletion.patch` — **critical fix**. Fixes the shared-block deletion bug in `_delete_block_from_postgres`. Scopes `BlocksAgents` delete to `agent_id` and only hard-deletes blocks that are globally orphaned. Required for multi-agent setups with shared blocks. See [Fixed section](#fixed) for details.
- **Server patch:** `patches/server_sync_delete_propagation.patch` — opt-in. Extends `GitEnabledBlockManager.sync_blocks_from_git` to remove postgres blocks whose source files no longer exist in git. Uses the already-existing `get_blocks_by_agent_async` and `_delete_block_from_postgres` helpers. Makes git actually behave as the source of truth: file deletions/renames now propagate to the postgres cache instead of leaving orphan blocks.
- **Server patch:** `patches/server_system_only_blocks.patch` — opt-in. Adds `LETTA_MEMFS_BLOCK_PATH_PREFIXES` env var to `letta/services/memory_repo/path_mapping.py`. When set (comma-separated list of prefixes, e.g. `system/`), only matching paths map to block labels; other `.md` files in the repo remain filesystem-only content. Unset / empty = upstream behavior unchanged. `skills/**/SKILL.md` always allowed through.
- Configuration Reference entry for `LETTA_MEMFS_BLOCK_PATH_PREFIXES`.
- `patches/README.md` now documents all three server patches with apply-order snippet and a note that they are independent (apply any subset).
- **Server patch:** `patches/server_memory_sync_endpoint.patch` exposes `POST /v1/agents/{agent_id}/memory/sync-from-git`, wrapping Letta's existing `GitEnabledBlockManager.sync_blocks_from_git` over HTTP. Optional `?recompile=true` chains a system prompt rebuild. Tested against letta-server 0.16.7.
- `patches/README.md` — documents both patches, their purpose, apply order, and tested versions.
- README section **Where the server actually stores memfs** explaining the bare-repo path and why fetches must target it with `--git-dir`.
- README **Known upstream Letta behaviors** table now marks additive-only sync and all-`.md`-mapping as "fixed by opt-in patch in this repo"; UTF-8 binary decode remains documented as a workaround (repo is intended to be markdown-only).
- README troubleshooting entries for additive-only sync, UTF-8 crash on binary files, and unexpected block creation for non-`system/` `.md` files.
- README **Acknowledgements** section crediting [@PurposeUnknown](https://github.com/PurposeUnknown) for the architectural finding that corrected the sync path.
- README section **Enabling MemFS for Your Agent** covering `/memfs enable` and the `git-memory-enabled` tag.
- README architecture note aligning the patch's model with Letta's own "git is source of truth, postgres is cache" framing (`letta/services/block_manager_git.py`).
- `docker-compose.override.yml`: named volume on `/root/.letta/memfs` so server state survives `docker compose down`.
- Troubleshooting entries for stale `<projection>` paths, `refusing to merge unrelated histories`, and "Memfs not enabled" on session init.
- Status row: tested with persistence volume (named volume / bind mount) and multi-machine sync (add/remove file on machine A → sync-from-git on machine B reflects change).

### Changed
- Single-machine deployments note is now surfaced at the top of the Server-Side Sync section — most self-hosted users only need step 2 (the sync endpoint), not the pull step.
- Installation is now 8 steps covering both client and server patches (was 5, client-only).
- Configuration Reference adds `LETTA_MEMFS_LOCAL` and the new API route.

### Removed
- `daemon` branch pattern from all examples and docs — `main` is now the sole documented branch

### Fixed (prior pass)
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
