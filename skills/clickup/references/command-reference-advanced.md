# cu command reference: tags, docs and the private frontdoor

Tags, Docs, everything past the public API, the ACL endpoint and the read-back traps. Split out of [`references/command-reference.md`](command-reference.md).

### Docs (ClickUp Docs v3 API)

| Command                                                | Purpose                            |
| ------------------------------------------------------ | ---------------------------------- |
| `cu docs list [--json] [--workspace <id>]`        | List docs in workspace             |
| `cu docs search [--creator <userId>] [--parent <id> --parent-type LIST] [--archived] [--deleted] [--limit <n>] [--cursor <c>]` | One page of the v3 doc search, with its filters |
| `cu docs info <docId> [--json]`                   | Doc metadata (parent, visibility)  |
| `cu docs pages <docId> [--json]`                  | List pages in a doc (with nesting) |
| `cu docs get <url\|pageId> [--doc <id>] [--json]` | Get page content                   |
| `cu docs create <docId> --name "..." [options]`   | Create new page                    |
| `cu docs update <url\|pageId> [options]`          | Update page content                |

**Doc create/update options**: `--content <text>`, `--file <path>`, `--name <title>`, `--mode replace|append|prepend`, `--parent <pageId>`, `--workspace <id>`, `--json`

Supports stdin piping:

```bash
cat file.md | cu docs update <pageId> --doc <docId>
```

Accepts full ClickUp URLs or page IDs for `get` and `update` commands.

**A Doc IS a view, and that is how you rename or delete one.** The docs API has no
rename and no delete: `PUT`/`DELETE` on `/v3/workspaces/{ws}/docs/{docId}` both answer
`405 method not allowed`, and pointing a page write at the doc id answers `403`. The doc
id is also a view id (`cu view get <docId>` returns `"type": "doc"`), so the v2 view
endpoints do both jobs (verified 25 Aug 2026):

```bash
cu view update <docId> --name "New Doc Title"   # rename a Doc
cu view delete <docId> --yes                    # delete a Doc
```

The doc TITLE lives there; page titles stay on `cu docs update <pageId> --name`.

**Never write a markdown table into a Doc page.** ClickUp renders it as a real table
block that is unreadable on mobile; use a bulleted list with a bold lead-in.
See `references/doc-authoring.md`.

**Moving a page to a new parent is a PUT on the page, not a doc-level call**, and which
credential it needs depends on whether the move stays inside the same doc:

```bash
curl -s -X PUT -H "Authorization: $TOK" -H "Content-Type: application/json" \
  -d '{"parent_page_id":"<destPageId>"}' \
  "https://api.clickup.com/api/v3/workspaces/<teamId>/docs/<docId>/pages/<pageId>"
```

A reparent within the same doc needs only the API token, and carries the whole
child subtree with it (children still point at the moved page). Moving a page to a
**different** doc is not the same call: it goes through the frontdoor JWT session
instead of the plain API token. Verify a reparent with a GET on the same page and
check `parent_page_id`.

### Beyond the public API (private frontdoor)

Needs a session from `cu net capture`; the `pk_` token does not work here.

| Command                                          | Purpose                                     |
| ------------------------------------------------ | ------------------------------------------- |
| `cu template center [--kind <k>]`           | Every saved template, by kind               |
| `cu template save <taskId> [--replace]`     | Save a task template, or re-save in place   |
| `cu dashboards`                             | Every dashboard in the workspace            |
| `cu statuses [--grep <p>]`                  | Every status defined anywhere               |
| `cu space statuses <id> [--set <spec>]`     | Read or replace a space's statuses          |
| `cu list statuses <id> [--set <spec>]`      | A list's own statuses; `--inherit` undoes   |
| `cu agents`                                 | Every Super Agent                           |
| `cu agents get <agentViewId>`               | One agent, `agent_config` included          |
| `cu automations --list <id> [--active <a>]` | Automations on a list                       |
| `cu automations count <listId>`             | Active and inactive counts                  |
| `cu automations catalog [--grep <p>]`       | Every automation trigger and action         |
| `cu automations apply-template <listId>`    | Task created → apply a template, replaced   |
| `cu automations delete <uuid>`              | Delete one rule                             |

`--set` entries are `name[:type][:#color]`, in order. The first defaults to `open`, the
last to `closed`, the rest to `custom`. A status left out is removed.

Template kinds use ClickUp's internal names: `task`, `subcategory` (list),
`project` (folder), `doc`. `--active` takes `ACTIVE`, `INACTIVE` or `ALL`.

### Private API capture (`cu net`)

The public API has no templates, automations, dashboards, space statuses or
Workload capacity. These commands record the calls ClickUp's own web app makes to
its private frontdoor API, then replay them without a browser. Full workflow and
guard rails: the `clickup` skill's browser mode.

| Command                                              | Purpose                                        |
| ---------------------------------------------------- | ---------------------------------------------- |
| `cu net capture <url> [--boot N]`                | Reload and record the whole boot sequence      |
| `cu net record [url] [--seconds N] [--settle N]` | Open a URL, inject the recorder, log while you click |
| `cu net drain [--out <file>]`                    | Append everything captured since the last drain |
| `cu net summarize <log> [--method M] [--grep P]` | Fold the log into distinct endpoints           |
| `cu net show <log> --index N [--secrets]`        | One call in full: headers and both bodies       |
| `cu net auth [--from <log>]`                     | Show the stored session headers and their expiry |
| `cu net replay <log> --index N [--allow-write]`  | Re-issue a recorded call from the terminal      |

**Recording only captures what happens while it runs.** The page must be clicked
through during the window, or the log comes back empty.

**`net replay` refuses writes by default.** POST, PUT, PATCH and DELETE need
`--allow-write`, because a replay is a real mutation on a live workspace. Read the
body with `net show` first.

**The captured bearer is the ~48h frontdoor JWT.** `net auth` prints its true `exp`
and `net replay` stops once it passes. Re-record to refresh; nothing auto-renews.

## Sharing a private object with a member

The v2 API cannot do it: there is no `POST /list/{id}/member`, and the guest endpoint needs
Enterprise. The **v3 ACL endpoint** can, with an ordinary personal `pk_` token, and it is wrapped:

```bash
cu acl set <objectId> --team <wsId> --object-type list --user <id> --permission edit
```

Underneath: `PATCH /api/v3/workspaces/{ws}/{object_type}/{object_id}/acls` with
`{"entries":[{"kind":"user","id":"<userId>","permission_level":4}]}`.

- **`kind` is required** (`"user"` or `"group"`). Omitting it returns the misleading
  `ACL_029 "Invalid group or user ID"`.
- `id` is a string. `permission_level` is 1 read, 3 comment, 4 edit, 5 create, and **null removes
  access**.
- `object_type` covers `list`, `folder`, `space`, `doc`, `view`, `task`, `dashboard` and more.
- PATCH **merges** entries rather than replacing them. Assigning a user to a task requires that they
  already have list access, or v2 answers `ITEM_087`.

**Making a list `private: true` drops workspace admins' implicit access**, the owner included, and
each one has to be re-granted explicitly. Read effective viewers with v2 `GET /list/{id}/member`;
the v3 `/acls` path is PATCH-only and GETs 405. A list showing nearly every workspace member is not
private.

Two shell notes for zsh: `UID` is read-only, and an unquoted `$VAR` does not word-split, so use
`${=VAR}`.

**Recurring tasks are UI-only.** The API exposes no `recurring` field on a task, and there is no
bulk-recurrence UI either.

## Reading a task back without being lied to

- **`cu task get <id>` truncates the description** in pretty mode, ending in `... (truncated)`.
  `--markdown` gives the full raw body, and it is the only reliable way to verify a description
  write.
- **ClickUp strips literal `#` and `###` heading markers on markdown ingest**, so grepping a
  readback for `### Acceptance criteria` gives a false negative. Verify by `--markdown` round-trip
  and length, not by header grep.
- **Pipe auto-detection is unreliable**: a piped `task get` or `tasks` still prints the pretty
  table. Pass `--json` or `--markdown` explicitly when scripting.
- **`--json` omits `checklists`, subtasks and some relations.** Read those from
  `GET /api/v2/task/{id}` directly, where `checklists[].resolved` and `.unresolved` are integer
  counts and the items are in `checklists[].items[]`.
- Bulk-create a checklist with
  `cu task checklist create <id> --name "..." --items-json "$(... | jq -R -s 'split("\n")|map(select(length>0))')"`.

## Editing a view's filters

`PUT /api/v2/view/{id}` must echo back `name`, `type`, `grouping`, `divide`, `sorting`, `filters`,
`columns` and `settings`, or the omitted fields reset.

A filter entry takes the exact shape
`{"field":"assignee","op":"ANY","determinor":null,"idx":0,"values":[<userId int>]}`. **`values` must
be integers**: string values are silently dropped and the `fields` array comes back empty.
`determinor` and `idx` are required keys. Scalar props like `show_closed` persist fine; only
`fields` is picky.

**`GET /api/v2/view/{id}/task` does not reliably honour multiple AND'ed filter fields.** One
assignee filter works and returns the correct reduced count, but adding a second field makes the
endpoint ignore the first and return nearly everything. Apply one filter through the API and add the
rest in the UI. The endpoint paginates 30 per page, so loop until `last_page: true`.

The v3 view endpoints (`/api/v3/workspaces/{ws}/views/{id}`) return 404 and do not exist.

## The AI Notetaker's call docs are invisible to `cu`

Each call transcript is written as a **standalone doc named after the calendar event**, not as a
page appended to the curated calls doc. Those docs are owned by and private to the account that
sat in the meeting, while `cu` authenticates as the workspace's shared token and has no
`--token` or env override, so `cu docs list` and `cu docs pages` return a clean "not there" for
every fresh transcript. **Do not conclude a transcript is missing from a `cu` result.**

```bash
TOK=$(jq -r '.tokens[1].token' ~/.config/clickup/config.json)
curl -s -H "Authorization: $TOK" \
  "https://api.clickup.com/api/v3/workspaces/<team-id>/docs?limit=100" \
  | jq -r '.docs[] | "\((.date_created/1000|floor|strftime("%Y-%m-%d %H:%M")))  \(.id)  \(.name)"' | sort | tail
curl -s -H "Authorization: $TOK" \
  "https://api.clickup.com/api/v3/workspaces/<team-id>/docs/<docId>/pages" \
  | jq -r '.[] | "\(.name)\n\(.content)"'
```

Two shapes that produce false negatives: `date_created` is already a number and `strftime` needs an
integer, so `(.date_created/1000|floor)`; and the pages endpoint returns a **bare array**, not
`{pages: [...]}`.

The doc body carries Attendees, Overview, Key Takeaways, Next Steps and Key Topics, plus an `.mp4`
recording link. The attendees listed there are the real participants and can exceed the calendar
guest list.

