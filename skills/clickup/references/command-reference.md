# cu command reference

Every command the CLI exposes, with its flags and the shape it returns. `SKILL.md` holds setup, the output modes and the traps; this is the lookup table. Views, custom fields, tags, docs and the private-frontdoor commands split out to [`references/command-reference-fields.md`](command-reference-fields.md) and [`references/command-reference-advanced.md`](command-reference-advanced.md).

## Command Reference

### Profiles, and the shorthand for IDs

**No command needs an API key on the command line.** A profile bundles its
tokens with the workspace they point at; `-p <name>` picks one for a single run,
`CU_PROFILE` for a shell, `cu profile use` for good.

```bash
cu profiles                       # which exist, which is active
cu -p personal tasks --list 901   # one run against another account
cu profile add work --token pk_... --team <team-id> --also-token "me:pk_..."
```

`--also-token name:pk_...` builds the fallback chain: a 401 on the first token
retries with the next. That is what makes a shared team token and a personal one
cover each other, and `cu bulk *` walks the chain per task.

**Every ID argument takes four shorthands**, and they are profile-scoped:

| Form | Means |
| --- | --- |
| `3` | row 3 of the last task listing (`cu tasks`, `cu sprint`) |
| `this` or nothing | the active task from `cu start` |
| `@alias` | a favourite (`cu favorite add task <id> bug`) |
| `sprint:current` | the current sprint's list, wherever a list ID goes |

The `#` column and `--pick` only appear on a TTY. Piped or driven by an agent
the rows are identical minus the column, so parsers never see a moving field.

**Short IDs are rewritten by every listing.** `cu tasks --list A` then
`cu tasks --list B` then `cu done 3` closes the third task of B. Re-list before
trusting a number you did not just see.

### Sprints

`cu sprint` and `cu sprints` find the folder whose name carries "sprint",
"iteration", "cycle" or "scrum", then the list whose dates cover today. No
match: pin one with `cu favorite add sprint-folder <folderId> sprints`. Between
sprints it picks the next to start; after the last, the most recent to end.

### Bulk edits

```bash
cu bulk status review --tasks 1,2,5
cu bulk move --list sprint:current --tasks @bug,86cb9uff6
```

Each row reports `ok` or `failed: <reason>` and the batch always finishes; the
exit code is non-zero if any row failed. Never a silent partial write.

### Hierarchy Navigation

| Command                                                 | Purpose                  |
| ------------------------------------------------------- | ------------------------ |
| `cu workspaces [--json]`                           | List all workspaces      |
| `cu spaces [--json] [--team <id>]`                 | List spaces in workspace |
| `cu folders --space <id> [--json]`                 | List folders in a space  |
| `cu lists [--space <id>] [--folder <id>] [--json]` | List lists               |
| `cu hierarchy [--json] [--team <id>]`              | Full workspace tree      |
| `cu members [--json] [--team <id>]`                | List team members        |
| `cu folder move <id> (--space <id> \| --parent-folder <id>)` | Move or nest a folder |

**`folder move` is the only way to re-parent a folder.** `folder update` renames and
nothing else. `--parent-folder` nests the folder as a subfolder and wins over `--space`;
it needs Subfolders enabled for the workspace, and ClickUp rejects the call otherwise.
Add `--type-map-json '{"<srcTypeId>": destTypeId}'` when the destination lacks a task
type the folder uses, or the move 400s.

**`cu hierarchy` is not the workspace.** It walks only the spaces the token owns, so
a space reached through folder sharing is absent and `GET /space/{id}` on it 401s. On
17 Aug 2026 one workspace's `hierarchy` printed 5 of 8 spaces and missed 636 tasks,
the CRM space among them; `cu shared` names some of their lists but no spaces. Never scope a
workspace-wide sweep off it. `GET /team/{id}/task?page=N&subtasks=true` returns every
task the token can see, shared ones included, and is the only reliable sweep. It is a
paged search index though, so counts drift between calls: verify a write per task.

### Tasks

**List & Search**:

```bash
cu tasks [--list <id>] [--assignee <ids...>] [--status <statuses...>] [--closed] [--subtasks] [--page <n>] [--json]
cu tasks query [--list <id> | --lists <ids> | --folder <ids> | --space <ids>] [--status <names>] \
  [--assignee <ids>] [--tag <names>] [--watcher <ids>] [--type <typeIds>] [--fields-json '<json>'] \
  [--parent <taskId>] [--created-after|--created-before|--updated-after|--updated-before|--due-after|--due-before <date>] \
  [--order-by id|created|updated|due_date] [--reverse] [--closed] [--subtasks] [--page <n>] [--json]
```

`tasks query` takes every filter Get Tasks and Get Filtered Team Tasks accept. `--type` is the
custom task type (`0` task, `1` milestone, the rest from `cu workspace task-types`), and
`--fields-json` is the API's custom field filter, for example
`[{"field_id":"…","operator":"IS NOT NULL"}]`. `--watcher` only works with `--list`, and
`--parent` only without it: Get Tasks has the watcher filter, the workspace-wide endpoint has
the parent one.

A comma list in `--status`, `--assignee` or `--folder` matches any of them. Before
29 Sep 2026 it was sent as one comma-joined value, so `--status "to do,done"` returned
nothing at all.

**A "live" filter is `status.type not in (closed, done)`, never `status != closed`.**
Custom statuses like `inactive`, `lost`, `no show`, `sent` and `complete` all carry
`type: done` without being named "closed", so filtering on the literal status name
alone counts finished work as open.

**Single Task**:

```bash
cu task get <taskId> [--json]           # Full task details
cu task create --list <id> --name "..." [options]
cu task update <taskId> [options]
cu open <taskId>                        # Open in browser
cu task estimate set <taskId> --estimate <userId>:<duration>   # per-assignee estimate
```

**`--time-estimate` and `task estimate set` are different fields.** `task update
--time-estimate 6h` writes the task's single total. `task estimate set --estimate
183:4h --estimate 204:2h` writes ClickUp's per-assignee estimates and merges into what
is already there; `--replace` drops every assignee you did not name. `unassigned` is a
valid user ID.

**The user must already be assigned to the task**, or ClickUp answers a bare
`400 Invalid Request` with no hint. Run `cu task update <id> --add-assignee <userId>`
first. The endpoint is Business plan and above, caps at 10 estimates per call, and
`unassigned` is the one exception to the assignment rule.

**Create options**: `--description <text>`, `--description-file <path>`, `--markdown`, `--status`, `--priority` (urgent/high/normal/low), `--assignee <ids...>`, `--tag <tags...>`, `--json`

**Update options**: `--name`, `--description <text>`, `--markdown`, `--status`, `--priority`, `--add-assignee <ids...>`, `--remove-assignee <ids...>`, `--archived`, `--json`

**Archiving is not a status.** A list has one done status, and closing a task that was
never done says it shipped. `cu task update <id> --archived` takes it out of every view
and off `cu tasks`; `--archived=false` brings it back.

**`--markdown` is a boolean, not a value flag.** It only says "treat the
description as markdown"; the content always goes in `--description`. Writing
`--markdown "$(cat body.md)"` is accepted silently and creates the task with an
**empty** description, because the body is consumed as the flag's value.
`--description-file` exists on `create` only, not on `update`.

```bash
cu task create --list <id> --name "..." --markdown --description "$(cat body.md)"
cu task update <taskId> --markdown --description "$(cat body.md)"   # no --description-file here
```

**`task get --json` omits the description entirely** and the plain text output
**truncates** it with a `... (truncated)` marker, so editing from either silently
deletes everything past the cut. `--markdown` prints the full raw description and
nothing else, which is what an edit-in-place needs:

```bash
cu task get <taskId> --markdown > body.md      # full, untruncated
cu task update <taskId> --markdown --description "$(cat body.md)"
```

### Comments

| Command                                                          | Purpose            |
| ---------------------------------------------------------------- | ------------------ |
| `cu comments list <taskId> [--json]`                        | List task comments |
| `cu comments add <taskId> --text "..." [--notify] [--json]` | Add a plain-text comment |
| `cu comments add <taskId> --from <file.json>`               | Add a formatted comment |

**`--text` is plain text and ClickUp renders it verbatim.** No headings, no bold, no
lists, and every newline you typed becomes a hard break in the reader's width. Markdown
posted this way arrives with `##` and `**` sitting in it as literal characters, wrapped
mid-sentence on every line. Use `--text` for one short sentence and nothing else.

Anything with structure goes through `--from`, which takes a **Quill Delta** body:
inline formatting rides on the text op, line formatting rides on the trailing newline.
Never hard-wrap a paragraph. Full format, the op-builder helpers, how to delete a
comment you already posted, and the description `--markdown` trap:
[`references/comment-authoring.md`](comment-authoring.md).

### Chat

| Command                                                | Purpose                       |
| ------------------------------------------------------ | ----------------------------- |
| `cu chat channels [--workspace <id>]`             | List channels and DMs         |
| `cu chat messages <channelId>`                    | Read a channel                |
| `cu chat send <channelId> --content "..." \| --file <f>` | Post a message           |
| `cu chat tagged <messageId>`                      | Who a message actually pinged |

**A mention is a link, not an `@name`.** Write `[@Sam](#user_mention#12345678)`; a
group is `[@everyone](#user_group_mention#<uuid>)`. ClickUp resolves the id in the link
target and ignores the label, so the label can be a short name. It resolves **only for a
member of that channel**: a mention of someone outside it renders as plain text and pings
nobody. Prove it with `cu chat tagged <messageId>` after sending, which lists who was
really tagged.

### Time Tracking

```bash
cu time [--start-date <ms>] [--end-date <ms>] [--assignee <id>] [--json]
```

