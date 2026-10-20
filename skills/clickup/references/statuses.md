# Custom statuses on a list, folder or space

The public API can't edit statuses (`PUT /v2/space` answers 200 and keeps the old array). The web
app's own routes can, and `cu statuses` drives them on the captured session. Run `cu net capture`
against the target workspace first. The session picks the workspace, not `CU_TEAM_ID`.

```
cu statuses get --list <id>              # also --folder, --space
cu statuses set --list <id> --file statuses.json [--map "old=new,..."] [--dry-run]
```

`statuses.json` is `[{"name": "à faire", "type": "open", "color": "#87909e"}, ...]`, or
`{"statuses": [...]}`. `type` is `open`, `custom`, `done` or `closed`. The first entry defaults to
open, the last to closed, the rest to custom. `color` is optional. Names keep their case and accents.

## The routes

Verified 20 Oct 2026 on the demo workspace, frontdoor host from the captured session.

| Scope | Method and path | Body |
| --- | --- | --- |
| List | `PUT /hierarchy/v1/subcategory/{id}` | `{override_statuses: true, statuses: [...]}` |
| Folder | `PUT /hierarchy/v1/category/{id}` | same |
| Space | `PUT /hierarchy/v1/project/{id}` | `{statuses: [...]}` |
| Inherit again (list, folder) | same path | `{override_statuses: false}` |

Each status is `{id?, status, type, orderindex, color}`. Reads are a GET on the same path:
`override_statuses` and `statuses`. Folder and space writes don't move tasks: a folder whose lists
hold tasks in a dropped status answers `conflict: true`, so set those lists one by one.

## What the server enforces

- **One `closed` status is required**, and only one. Without it: `400 STATUS_003 Closed is a
  required status type and there can only be one`. A second end state is `done`. So a list that
  wants "paid" as its end state makes it `closed`, not `done`.
- **Tasks in a dropped status block the write.** The answer is `200` with `conflict: true` and
  nothing changes. `cu statuses set` handles it for a list: it writes the new set plus the old
  statuses as `custom`, moves each task, then writes the final set. `--dry-run` prints the moves
  first.
- **A list's own statuses get new ids** (`sc{listId}_...`) on the first write. Read again before a
  second write and match by name, which the command does.
- Where a task goes: same name stays, an open one goes to the first status, done and closed ones
  go to the target's `done` (else `closed`), a custom one goes to the first custom. `--map` overrides
  by old status name.
- Subtasks are in the list, so they move too. Archived tasks aren't returned and would still block.

Code: `api/statuses-api.ts` (routes, body, migration) and the `statuses get|set` commands in
`cli/src/index.ts`.
