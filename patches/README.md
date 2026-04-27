# Patches

Four patches — one client, three server. The client patch + the sync-from-git endpoint are required; the other two server patches are opt-in hardening for known upstream behaviors that hurt multi-machine users.

**Apply order on a fresh `letta-source` checkout:**

```bash
patch -p1 < patches/server_memory_sync_endpoint.patch       # required
patch -p1 < patches/server_sync_delete_propagation.patch    # opt-in
patch -p1 < patches/server_system_only_blocks.patch         # opt-in
```

All three server patches modify different files (or different regions of `block_manager_git.py`) and are independent of each other — you can apply any subset.

## `memoryGit.ts.patch` — client side (letta-code)

Three additive hunks to `src/agent/memoryGit.ts`:

1. `getGitRemoteUrl()` consults `LETTA_MEMFS_GIT_URL` and substitutes `{agentId}` before falling back to the server's `/v1/git/` proxy.
2. `configureLocalCredentialHelper()` short-circuits when `LETTA_MEMFS_GIT_URL` is set (external hosts typically embed auth in the URL or run their own credential flow).
3. `cloneMemoryRepo()` ensures `origin` points at the resolved URL, updating or adding the remote if needed (handles migration from the sidecar to an external host on an existing clone).

No existing logic is removed.

### Applying

```bash
cd /path/to/letta-code
patch -p1 < /path/to/letta-external-memfs/patches/memoryGit.ts.patch
bun run build
```

## `server_memory_sync_endpoint.patch` — server side (letta-server)

Adds one new route and one import line to `letta/server/rest_api/routers/v1/agents.py`. No existing code modified.

### What it adds

```
POST /v1/agents/{agent_id}/memory/sync-from-git
Query params:
  recompile: bool (default false) — chain rebuild_system_prompt_async after sync
Returns: 200 List[Block] on success
         409 if server isn't running GitEnabledBlockManager
         422 on invalid agent_id format
```

### Why it's needed

Letta's `GitEnabledBlockManager` already implements `sync_blocks_from_git(agent_id, actor)` (`letta/services/block_manager_git.py`). Internally, it reads the server's backing git repo and refreshes the postgres cache block-by-block. Stock Letta calls it only from the `/v1/git/` push-receive hook (`letta/server/rest_api/routers/v1/git_http.py::_sync_after_push`), which the client-side patch bypasses.

This patch exposes the existing primitive over HTTP so an external sync process can trigger the files → postgres cache materialization that normally fires on receive.

### Applying

```bash
cd /path/to/letta-source    # your letta-server checkout
patch -p1 < /path/to/letta-external-memfs/patches/server_memory_sync_endpoint.patch
# then rebuild your docker image per your build flow
```

## `server_sync_delete_propagation.patch` — opt-in hardening

Extends `GitEnabledBlockManager.sync_blocks_from_git` to also remove orphan postgres blocks — i.e., blocks whose source files no longer exist in the agent's git repo. Stock `sync_blocks_from_git` is additive-only: it upserts one block per file in git but never scans postgres for blocks to delete. After a file delete or rename in git, the old block label sticks around until manually removed via the API.

With this patch, after the existing upsert loop, the sync computes `{postgres labels} − {git labels}` and calls the existing `_delete_block_from_postgres` helper for each orphan. No new helpers, no new dependencies. `get_blocks_by_agent_async` and `_delete_block_from_postgres` already exist in 0.16.7.

Scope: ~15 lines added inside `sync_blocks_from_git`. No effect on agents that aren't using `GitEnabledBlockManager`.

### Applying

```bash
cd /path/to/letta-source
patch -p1 < /path/to/letta-external-memfs/patches/server_sync_delete_propagation.patch
```

## `server_system_only_blocks.patch` — opt-in filter

Adds an env-gated path-prefix filter to `letta/services/memory_repo/path_mapping.py`. Upstream maps every `.md` file under an agent's memfs repo to a postgres block label (with `skills/**/SKILL.md` specially handled). If you want to use the memfs repo as backup storage without promoting every `.md` file to a block, set:

```
LETTA_MEMFS_BLOCK_PATH_PREFIXES=system/
```

Only paths starting with one of the configured prefixes (comma-separated, e.g. `system/,users/`) produce a block label. `skills/**/SKILL.md` is always allowed through so existing skill behavior is preserved regardless of the filter. Unset / empty env var = upstream behavior unchanged.

### Applying

```bash
cd /path/to/letta-source
patch -p1 < /path/to/letta-external-memfs/patches/server_system_only_blocks.patch
```

Then set `LETTA_MEMFS_BLOCK_PATH_PREFIXES` in your compose or shell env to opt in.

## `server_fix_shared_block_deletion.patch` — critical bug fix

Fixes a bug in `letta/services/block_manager_git.py::_delete_block_from_postgres` where deleting a block from one agent would also delete it from all other agents sharing that block.

**The bug:** Line 178 in the original code deletes from `BlocksAgents` without scoping to `agent_id`:
```python
await session.execute(delete(BlocksAgents).where(BlocksAgents.block_id == block.id))
```

This strips the block from **every attached agent**, not just the one being deleted. In multi-agent setups with shared blocks (e.g., 12-way or 15-way attachments), syncing one agent silently removes the block from all readers.

**The fix:** Scope the delete to the specific agent, then only hard-delete the block if it's globally orphaned:
```python
# Detach this agent only
await session.execute(
    delete(BlocksAgents).where(
        BlocksAgents.agent_id == agent_id,
        BlocksAgents.block_id == block.id,
    )
)
await session.flush()

# Only hard-delete if globally orphaned
remaining = await session.execute(
    select(func.count()).select_from(BlocksAgents).where(BlocksAgents.block_id == block.id)
)
if remaining.scalar() == 0:
    await block.hard_delete_async(db_session=session, actor=actor)
```

### Impact

- **Single-agent setups**: No change in behavior
- **Multi-agent setups with shared blocks**: Blocks remain intact across all agents
- **Fixes a latent bug in upstream Letta**: The same bug exists in `delete_block_async` (line 348) when users explicitly delete blocks on memfs+shared-blocks setups

### Applying

```bash
cd /path/to/letta-source
patch -p1 < /path/to/letta-external-memfs/patches/server_fix_shared_block_deletion.patch
```

**Note**: This patch should be applied **after** `server_sync_delete_propagation.patch` if you're using both, as the delete propagation patch calls `_delete_block_from_postgres`.

## Tested Against

| Component | Version |
|---|---|
| letta-server | 0.16.7 (commit `a1333365`) |
| letta-code | 0.19.6+ |

Patches use diff context rather than hash-pinning, so they should apply across minor version bumps unless Letta actively edits the surrounding lines. Always `git apply --check` (or `patch --dry-run`) before applying.

## Upstream

The server patch is self-contained and would be useful to any Letta deployment running `GitEnabledBlockManager`, not just the external-memfs pattern — it exposes an existing internal primitive that replaces hand-rolled PATCH loops for block cache rebuilds. Candidate for an upstream PR to Letta. If accepted, this patch disappears.

Draft commit message, PR title, and PR body for the upstream submission are in [`upstream-pr-draft.md`](upstream-pr-draft.md).
