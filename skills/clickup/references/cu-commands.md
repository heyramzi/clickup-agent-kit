# The cu CLI: the traps

`cu --help` and `cu <command> --help` list every command and flag, and the binary wins over
anything here. This file covers what help can't tell you. Fields, tags and views are in
[`fields.md`](fields.md), docs in [`bulk-and-docs.md`](bulk-and-docs.md).

## Auth and output

- Config is `~/.config/clickup/config.json`, shaped `{defaultProfile, profiles: {<name>: {tokens: [...]}}}`.
  Reading `d['tokens']` raises `KeyError: 'tokens'`, which looks like a broken config. Override
  with `CU_API_TOKEN` / `CU_TEAM_ID`, or `-p <name>` for one run.
- Fallback is per call and isn't a retry: a command runs on the first token and stops. A
  `401 ACCESS_081` means the other token owns the object. It doesn't mean the op is forbidden.
- A member token is missing whole scopes. Tag writes need `manage_tags` (`tag create` 401s
  `ACCESS_081`, `tag delete` 401s `ACCESS_016`), and custom-field writes 403 `FIELD_220` with the
  message "does not have permission to create fields", which reads like a plan limit but isn't one.
  Both go on the owner token. `cu` has no flag for it, so export the other token for one call:

  ```bash
  export CU_API_TOKEN=$(python3 -c "import json;d=json.load(open('$HOME/.config/clickup/config.json'));p=d['profiles'][d['defaultProfile']];print(p['tokens'][-1]['token'])")
  ```

- In a workspace 2 people built in, one token can't rebuild a space: deleting someone else's
  task is `401 ACCESS_081` and creating in their list is `401 ACCESS_064`. It's per object and not per list.
  Retry any 401 on the second token.
- A pipe doesn't switch off the pretty table reliably. Pass `--json` or `--markdown` when
  scripting. JSON is followed by a `help[...]` block, so parse the first line or use
  `json.JSONDecoder().raw_decode(text)[0]`. `cu tasks --json` is `{count, items}` and hides
  Closed tasks without `--closed`.
- An ID argument takes `3` (row 3 of the last listing), `this` (the task from `cu start`),
  `@alias` and `sprint:current`. Listings renumber each time, so re-list before `cu task update 3 --status <done status>`.

## Reading

- `cu hierarchy` doesn't cover the whole workspace. It walks only spaces the token owns. On 17 Aug 2026 it
  printed 5 of 8 spaces and missed 636 tasks. Sweep with
  `GET /team/{id}/task?page=N&subtasks=true`, and verify a write per task, because counts drift.
- A live filter is `status.type not in (closed, done)`, and `status != closed` is wrong because `inactive`,
  `lost`, `sent` and `complete` all have type `done`.
- `tasks query` takes `--due-before`, `--fields-json '[{"field_id":"...","operator":"IS NULL"}]'`,
  `--tag`, `--type` and more. `--watcher` needs `--list`; `--parent` needs no list. A comma list
  in `--status`, `--assignee` or `--folder` matches any of them.
- `task get` lies about the description. Pretty mode truncates it with `... (truncated)`,
  `--json` drops it, and `--json` also drops custom fields (`--fields` prints them), checklists
  and subtasks (read those from `GET /api/v2/task/{id}`). `--markdown` prints the full body and
  is the only safe base for an edit-in-place.
- ClickUp strips `#` heading markers on markdown ingest, so grepping a readback for `### Goals`
  gives a false negative. Compare `--markdown` length instead.
- `cu task members` is who can SEE a task; `cu task assignees <id>` is who's on it. Build a
  `--remove-assignee` list from the first and the old owner survives (26 Aug 2026: 15 of 157 tasks).
- An AI agent assignee has a NEGATIVE user id. A parser testing `line[0].isdigit()` drops it.
- Recurring tasks are UI-only. The API has no `recurring` field.

## Writing tasks

- Before `cu task create`, look for a stale task anywhere in the workspace and reuse it: move it to
  the right list, then revamp all of it (name, description, status, dates, fields, checklist, tags,
  assignee). Stale means genuinely obsolete and still open, like an idea that won't be made or a dead
  duplicate. A closed task isn't stale. It also needs no comments, no time logged, no attachment
  still needed, and it can't be a client task. Unsure, create a new one. It cleans the board in the
  same move.
- `description` is plain text and `markdown_content` renders markdown. Both return 200, so a
  `bold` body in `description` shows asterisks. In the CLI, `--markdown` is a boolean and the
  body still goes in `--description`: `--markdown "$(cat body.md)"` silently creates an empty task.
  `--description-file` and `--tag` exist on `create` only.
  `cu task update <id> --markdown --description "$(cat body.md)"`
- Dates are epoch milliseconds, everywhere. `--start-date 2026-09-14` is accepted and lands on
  the wrong day (start Sep 7 on 8 Aug 2026), and date custom fields reject it (`FIELD_017`). Pair
  the epoch with `--start-date-time false` / `--due-date-time false` for date-only, then read it
  back. A due date also comes back rounded unless the write sends `due_date_time: true`, so a sync
  that compares timestamps rewrites every task on every run.
- `--time-estimate` is the task's one total. `task estimate set <id> --estimate 183:4h` is per
  assignee, needs the user already assigned (else a bare `400 Invalid Request`), Business plan up,
  max 10 per call; `--replace` drops everyone you didn't name.
- Archiving isn't a status: `cu task update <id> --archived` (`=false` restores). A list has one
  done status, and closing a task that wasn't done says it shipped.
- A guest assigned to a task they can't see returns 200 and assigns nobody. 135 tasks said
  success and 3 points of a 25-point Workload showed. Share first
  (`cu guest share <guestId> --folder <id> --permission edit`, `cu list members <listId>`), then
  assign, and read assignees back off `cu tasks --list`.
- A list an automation writes to can't hold a sync. A Task-created rule renamed 17 mirrored
  tasks about 10 seconds after the create returned 200, and every later run read it as drift.
  Give a sync its own list, or create one task and read it back after a few seconds.
- `cu task checklist create <id> --name X --items-json '[...]'` bulk-creates a checklist.
- Moving a folder: `cu folder move` is the only way (`folder update` renames). `--parent-folder`
  needs Subfolders on; add `--type-map-json` when the destination lacks a task type the folder uses.

## Comments and chat

- `cu comments list` pages through every comment, 25 a call, newest first. `--limit <n>` stops sooner.

- `comments add --text` is verbatim plain text. A typed newline becomes a hard break and
  `##` shows literally. Use it for one short sentence. Anything structured goes through
  `--from comment.json` = `{"comment": [ops], "notify_all": false}`, a Quill Delta.
- Inline formatting rides on the text op (`{"text": "Title", "attributes": {"bold": true}}`); line
  formatting rides on the trailing `"\n"` op (`{"header": 2}`, `{"list": "bullet"}`). Inline:
  `bold italic underline strike code link`. Line: `header list blockquote code-block divider indent`.
  Mention: `{"type": "tag", "user": {"id": 123}, "text": "@Name"}`. Build the ops in a small script.
- Write each paragraph as one long line. A brief hard-wrapped at 90 columns arrived with a break
  every 90 characters. This applies to descriptions and Doc pages too.
- `cu comments delete <commentId>` removes a comment. `cu comments list --json` prints a banner before the JSON; read back with
  `GET /api/v2/task/<id>/comment | jq '.comments[0].comment[]'`.
- `cu chat channels` filters with `--is-follower`, `--include-closed`, `--with-message-since <date>`
  and `--channel-types CHANNEL,DM,GROUP_DM,ASK_AI`. `cu chat get` prints `latest_comment_at`.
- `cu chat messages` truncates to 120 chars with no author. `--full` plus `cu chat members` fixes
  both. A DM 404s on a token that isn't a party to it, so use the owner profile.
- A chat mention is a link: `[@Sam](#user_mention#12345678)`, group `[@everyone](#user_group_mention#<uuid>)`.
  It pings only a channel member. Prove it with `cu chat tagged <messageId>`.

## Time, custom IDs, nesting

- `cu time` with no dates is the last 30 days, and yours only. Narrow with `--task`, `--list`,
  `--folder` or `--space` (one of them), `--billable true|false`, `--assignee` and dates as
  `YYYY-MM-DD` (UTC, an end date covers the whole day).
- `--custom-ids` works on any command and reads task IDs like `PROJ-12` in your workspace
  (`CU_TEAM_ID`). It never guesses: without the flag an ID is a native one. A parent or
  link target in a body is always a native ID.
- `cu task create --links-to <taskId>` links the new task, and `--check-required-fields` makes
  ClickUp enforce required custom fields, which it skips on create by default.
  `cu task checklist item update --parent <itemId>` nests an item (`--parent ""` un-nests it).
- `cu docs pages --max-depth <n>`, `cu docs get --content-format text/plain`, and
  `cu docs create --sub-title <text>`.

## Sharing, roles, replies

- No `POST /list/{id}/member` in v2, and guests need Enterprise. `cu acl set <objectId> --team <ws>
  --object-type list --user <id> --permission edit` calls the v3 ACL endpoint with a personal token:
  `kind` is required (`ACL_029 "Invalid group or user ID"` otherwise), levels are 1 read, 3 comment,
  4 edit, 5 create, null removes, PATCH merges. A task assignee needs list access first (`ITEM_087`).
- `private: true` on a list drops admins' implicit access, owner included. Read viewers with
  `GET /list/{id}/member`; a list showing nearly everyone isn't private.
- `cu user update <userId> --admin[=false]` uses the frontdoor (roles 1 owner, 2 admin, 3 member,
  4 guest) because the public route 403s below Enterprise. It needs an owner/admin session
  (`401 ACCESS_009` otherwise); set `CLICKUP_FRONTDOOR_JWT` when the browser is signed in as a member.
- zsh: `UID` is read-only and an unquoted `$VAR` doesn't word-split (use `${=VAR}`).
- An agent's answer to a task mention is a threaded reply at `GET /api/v2/comment/<id>/reply`, about 90 seconds later. It isn't a
  comment, so a flat comment count doesn't mean it failed.
