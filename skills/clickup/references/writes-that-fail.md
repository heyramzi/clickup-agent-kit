# The writes that fail on their first honest attempt

Every entry is a run somebody already paid for. Read it before a write this skill has not
already made once.

Both verified 24 Aug 2026 while building a demo list.

**A dropdown value is an option index or a uuid, never the label.** `cu task field set <id>
--field <fieldId> --value "Email sent"` returns
`400 {"err":"Value must be an option index or uuid","ECODE":"FIELD_011"}`. Pass the
zero-based position instead, so the third option is `--value 2`. Reading the ids back is
harder than it should be: `cu fields list --json` flattens `type_config.options` to a
display string (`"Queued | Email sent | Replied"`), so the ids are not in the CLI's output
at all. Count the positions off that string, or call `GET /api/v2/list/{id}/field` when a
script needs the uuids.

**`--columns-json` must not list `name`.** ClickUp always renders the Name column, so
including `{"field":"name"}` creates a second one. The view then draws a duplicate empty
Name header and **every later header is offset by one column**, which reads as the custom
fields having no values when the values are simply sitting under the wrong labels. List
only the fields after Name:

    cu view create --list <id> --name Nurture --type list \
      --group-by "cf_<fieldId>" \
      --columns-json '{"fields":[{"field":"cf_<dateFieldId>","width":200,"hidden":false},{"field":"assignee","width":140,"hidden":false}]}'

**`cu view delete` prompts and is not idempotent in a pipeline.** It asks
`Delete <id>? (y/N)` on stdin; a script that does not answer leaves the view alive while
the next command in the same line still runs, which is how one run ended up with two views
of the same name. Pipe `yes |` into it, and re-read the list afterwards.

**A `403 FIELD_220` on `cu fields create` is the wrong token, not a missing capability.**
Measured 24 Aug 2026 creating a `url` field on a marketing list: the default priority-1
team token answers
`403 {"err":"User does not have permission to create fields in this location","ECODE":"FIELD_220"}`,
and the same request on the priority-2 owner token answers `200` with the new field. The
message reads as a plan limit or a UI-only surface and sent one run off to the browser skill
for nothing. **Custom-field writes go on the owner token**; `cu` has no flag to pick one, so
read it out of `~/.config/clickup/config.json` (`tokens[1].token`) and call the REST endpoint
directly:

```bash
OWNER=$(python3 -c "import json;d=json.load(open('$HOME/.config/clickup/config.json'));p=d['profiles'][d['defaultProfile']];print(p['tokens'][-1]['token'])")
curl -s -X POST "https://api.clickup.com/api/v2/list/<listId>/field" \
  -H "Authorization: $OWNER" -H 'Content-Type: application/json' \
  -d '{"name":"Brief","type":"url"}'
```

The config is **profile-shaped**: `{"defaultProfile": "...", "profiles": {"<name>": {"tokens": [...]}}}`.
A path reading `d['tokens']` raises `KeyError: 'tokens'`, which reads as a broken config rather than
as a wrong path.

**A field created on a list lives on that list only. `POST /v2/space/{spaceId}/field` creates it
once for the whole space**, which is what you want whenever more than one list has to carry the same
field and be read by one automation. It is undocumented and returns the normal `{"field": {...}}`
body; measured 26 Aug 2026 building a per-client billing space, where a per-list field would have
meant a different uuid per client and an automation that could not address them.

```bash
curl -s -X POST "https://api.clickup.com/api/v2/space/<spaceId>/field" \
  -H "Authorization: $OWNER" -H 'Content-Type: application/json' \
  -d '{"name":"Heures à facturer","type":"number"}'
```

The same endpoint takes a `drop_down` with `type_config.options` (`name`, `color`, `orderindex`) and
a `currency` with `type_config.currency_type`.

**Merging two custom fields is `cu fields merge`, not the Field Manager.** ClickUp's own Merge
action is Business Plus and Enterprise only; the CLI does it on any plan and is what this skill
uses: [`field-merging.md`](field-merging.md).

**Assigning a guest to a task they cannot see returns `200` and assigns nobody.** Measured
26 Aug 2026 loading a demo workspace: `cu task update <id> --add-assignee <guestId>` answered
success on all 135 tasks, and the guest came out assigned only on the one list they had been
shared into. The rest stayed unassigned, which showed up two steps later as a Workload row at
3 points instead of 25 and an `Unassigned` row holding the difference. A member sees every list
in the workspace and never hits this; a guest sees only what was shared. Grant the access first,
then assign:

    cu list members <listId>                                   # who can actually be assigned
    cu guest share <guestId> --folder <folderId> --permission edit

**Read the assignment back through the thing that consumes it**, not through the exit code. The
write is silently partial, so `cu tasks --list <id>` with an empty assignee column is the only
proof.

**`cu task members` answers who can SEE a task. `cu task assignees` answers who is on it.**
The two read alike and one run built its `--remove-assignee` list from the first, so the
previous owner survived on 15 of 157 tasks while the new one was added on top, and the
Workload view went on counting people who had been taken off the board. `cu task assignees
<id>` (2.6.0, added 26 Aug 2026 for exactly this) returns the real list with ids.

**An AI agent assignee carries a NEGATIVE user id.** `Jony - Invoice follow up AI` is
`-40578317`. A parser that tests `line[0].isdigit()` to find id rows
drops it in silence, so the agent stays assigned and the task keeps two owners.

**In a workspace two people have built in, one token cannot rebuild a space.** Deleting a task
somebody else created answers `401 ACCESS_081 / INSUFFICIENT_ACCESS`, and creating a task in a
list they own answers `401 ACCESS_064 / can_create_tasks`, per object rather than per list: the
demo workspace holds both directions, so a builder that reads 300 tasks and rewrites them dies
twice, halfway through, on different lists. Give the client a second token and retry any 401 on
it (the demo builder does exactly that), rather than switching
profiles by hand and re-running. Measured 18 Sep 2026 rebuilding DELIVERY and SALES.

**A list an automation writes to cannot hold a sync, and the collision is silent.** Seventeen
tasks mirrored onto MARKETING › Newsletter on 18 Sep 2026 all came back named
"Newsletter – September 2026": a Task-created automation on that list renames every new task
about ten seconds after it lands, long after the create returns `200` with the name you asked
for. Nothing in the API says the automation exists, so every later run read the name as drift
and rewrote it. **Give a sync its own list.** Before writing many tasks to a list somebody set
up by hand, create one, read it back after a few seconds, and compare.

**`due_date` comes back rounded unless the write also sends `due_date_time: true`.** ClickUp
stores a date-only due date at a fixed hour, so a sync that compares the timestamp it wrote
against the one it reads drifts on every run and rewrites every task forever.
