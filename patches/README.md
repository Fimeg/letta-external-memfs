# Patches

Two patches — one client, one server. Both are required for the external-memfs setup to fully work.

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

## Tested Against

| Component | Version |
|---|---|
| letta-server | 0.16.7 (commit `a1333365`) |
| letta-code | 0.19.6+ |

Patches use diff context rather than hash-pinning, so they should apply across minor version bumps unless Letta actively edits the surrounding lines. Always `git apply --check` (or `patch --dry-run`) before applying.

## Upstream

The server patch is self-contained and would be useful to any Letta deployment running `GitEnabledBlockManager`, not just the external-memfs pattern — it exposes an existing internal primitive that replaces hand-rolled PATCH loops for block cache rebuilds. Candidate for an upstream PR to Letta. If accepted, this patch disappears.

Draft commit message, PR title, and PR body for the upstream submission are in [`upstream-pr-draft.md`](upstream-pr-draft.md).
