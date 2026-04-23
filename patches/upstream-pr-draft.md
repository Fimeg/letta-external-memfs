# Upstream PR Draft — `sync_blocks_from_git` HTTP endpoint

Draft for submitting `server_memory_sync_endpoint.patch` as a standalone contribution to [letta-ai/letta](https://github.com/letta-ai/letta). The patch is useful to any deployment running `GitEnabledBlockManager`, not only the external-memfs pattern, which is why it's worth proposing independent of the rest of this repo.

---

## Suggested commit message

```
server: expose sync_blocks_from_git via POST /v1/agents/{id}/memory/sync-from-git

GitEnabledBlockManager.sync_blocks_from_git already implements the files
-> postgres block materialization that fires from the /v1/git/ push-receive
hook (_sync_after_push in git_http.py). Expose it as an HTTP endpoint so
external sync processes (webhooks, scheduled pulls, custom deployment
flows, recovery from drift) can trigger a cache rebuild without going
through a push cycle.

Returns 200 List[Block] on success, 409 when the server isn't running
GitEnabledBlockManager, 422 for invalid agent_id format. Optional
?recompile=true chains rebuild_system_prompt_async so callers can refresh
the compiled system prompt in the same round trip. Default false, since
recompile invalidates the provider KV cache and should be paid only when
system/ blocks actually changed.

No existing code paths are modified. One new route + one added import.
```

## Suggested PR title

```
Expose sync_blocks_from_git as POST /v1/agents/{id}/memory/sync-from-git
```

## Suggested PR body

---

### Summary

Exposes `GitEnabledBlockManager.sync_blocks_from_git` over HTTP as a new route:

```
POST /v1/agents/{agent_id}/memory/sync-from-git
  ?recompile: bool (default false)
```

Returns `List[Block]` on success, `409` if the server isn't configured with `GitEnabledBlockManager`, `422` on invalid `agent_id` format.

### Motivation

`sync_blocks_from_git` already exists at `letta/services/block_manager_git.py:564` and implements the files → postgres block materialization that `_sync_after_push` invokes internally after a successful `git-receive-pack`. Today, that's the only code path that calls it.

For deployments where the push-receive hook doesn't fire — external sync mechanisms, scheduled pulls, webhook-driven workflows, recovery from postgres/git drift — there's no public way to trigger the cache rebuild. Users today either hand-roll per-block `PATCH /v1/blocks/{id}` loops (which duplicate work the server already knows how to do and depend on the caller knowing the label→filename mapping), or reach into the server via `docker exec python -c "from letta.services.block_manager_git import ..."`.

This PR doesn't add logic. It adds a URL to an existing function.

### Use cases

- Self-hosted deployments where `git push` goes to an external host rather than `/v1/git/` (the push-receive hook doesn't fire).
- Any deployment where files on the server's backing memfs are modified outside the push path: filesystem syncs, restore from snapshot, git hook pulls, disaster recovery.
- Drift repair after detecting postgres/git divergence.

### Behavior

| Query param | Default | Effect |
|---|---|---|
| `recompile=false` | default | Rebuild the postgres cache only. Cheap. Changes are picked up in the system prompt on the next natural compaction. |
| `recompile=true` | | Also calls `rebuild_system_prompt_async(force=True)`. Invalidates provider KV cache. Use when callers actively need the prompt to reflect new `system/` content immediately. |

The default is `false` specifically so operators don't pay recompile cost on every sync — most syncs don't change `system/`-pinned blocks, and natural compaction handles the rest.

### Scope

Deliberately minimal:

- One new route in `letta/server/rest_api/routers/v1/agents.py`
- One added import (`Block` from `letta.schemas.block`)
- No changes to existing code paths, schemas, or behavior

### Testing

Smoke-tested against self-hosted letta-server 0.16.7:

- Invalid `agent_id` format → `422` (FastAPI path validator)
- Valid UUID4, no git repo for agent → `200 []` (empty list from the internal primitive)
- Server not using `GitEnabledBlockManager` → `409` (explicit guard)

Happy to add integration coverage matching `tests/integration_test_git_http_post_push_sync.py` style if reviewers prefer that before merge — the existing post-push sync tests already cover `sync_blocks_from_git` behavior; this PR is just a routing surface on top.

### Related context

For anyone reviewing: this came out of community work on an external-memfs pattern (https://github.com/Fimeg/letta-external-memfs) where the client-side push is redirected to an external git host, which causes the `/v1/git/` receive hook to never fire. The absence of this endpoint forces every such deployment to reinvent the block-rebuild path. The endpoint is independently useful regardless of whether that pattern is adopted.

---

## Branch / file layout for the PR

```
letta/server/rest_api/routers/v1/agents.py   # +51 / -1
```

Apply via:

```bash
cd /path/to/letta          # upstream clone
git checkout -b add-sync-from-git-endpoint
patch -p1 < /path/to/letta-external-memfs/patches/server_memory_sync_endpoint.patch
git add letta/server/rest_api/routers/v1/agents.py
git commit  # paste the commit message above
git push -u origin add-sync-from-git-endpoint
# then open PR via GitHub UI or gh pr create
```
