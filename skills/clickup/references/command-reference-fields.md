# cu command reference: views and custom fields

Views and custom-field CRUD, split out of [`references/command-reference.md`](command-reference.md) to stay under the length cap.

### Views

```bash
cu views list --space <id>             # Views on a space
cu views list --folder <id>            # Views on a folder
cu views list --list <id>              # Views on a list

cu view get <viewId>                   # Full details (columns, filters, grouping)
cu view create --list <id> --name "X" --type board
cu view update <viewId> --name "New Name"
cu view update <viewId> --columns-json '{"fields":[{"field":"assignee","hidden":false}]}'
cu view delete <viewId> [--yes]
```

System views (IDs like `4-SPACEID-28`) cannot be deleted. Only user-created views (IDs `8cbypq9-*`).

**A ClickApp that is off eats the value in silence.** `POST /task` accepts
`priority` on a space where the Priorities ClickApp is disabled, stores it, and
returns it on every read, so the CLI and the API both look correct. The web app
shows no flag and a view grouped on it puts every task under "No Priority", which
reads like the grouping is broken rather than the space. The same holds for Sprints,
Milestones and native points. Read the switches before you blame the view:

```bash
curl -s -H "Authorization: $TOKEN" "https://api.clickup.com/api/v2/space/<id>" \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['features'].keys())"
```

A missing key means the ClickApp is off, not that it defaults to on. Turn it on and
the values that were already stored appear at once, with no re-write of the tasks:

```bash
curl -s -X PUT -H "Authorization: $TOKEN" -H "Content-Type: application/json" \
  -d '{"features":{"priorities":{"enabled":true}}}' \
  "https://api.clickup.com/api/v2/space/<id>"
```

A custom field column needs its ID prefixed `cf_` (matching `groupBy`'s format), e.g. `{"field":"cf_d3ae6727-a428-42f8-8361-94a2d448e818","hidden":false}`. The bare field ID 404s with `"View Not Found"`, which reads like the view itself is missing rather than a field-reference format error. Built-in fields (`assignee`, `status`, `priority`, `dueDate`, ...) take no prefix.

### Custom Fields

```bash
cu fields list --list <id>                          # Fields + their dropdown options
cu fields list --list <id> --task-types             # + the task types each field is scoped to
cu fields folder|space <id> --task-types            # same at the folder / space level
cu fields workspace --task-types
cu fields create --list <id> --name "X" --type drop_down --options-json '["A","B"]'
cu fields update <fieldId> --list <id> --name "New name"
cu fields update <fieldId> --list <id> --add-options "Q3 2026,Q4 2026"
cu fields update <fieldId> --list <id> --rename-options "Declaracao mensal=Declaração mensal"

cu task field set <taskId> --field <fieldId> --value <optionId|index>
cu task field unset <taskId> --field <fieldId>
```

Reading, deduplicating and deleting fields across a whole workspace is four more
commands, on the private API (`cu net capture` first):

```bash
cu fields all [--type drop_down] [--level workspace|space|folder|list]
cu fields duplicates [--loose]                      # groups near-twins, survivor first
cu fields merge --into <keepId> --from <id,id> --dry-run
cu fields delete <fieldId> --list <id>              # or --folder / --space
```

The merge moves every task value onto the kept field server-side and cannot be
undone. Workflow, refusal codes and what a merge breaks:
[`references/field-merging.md`](field-merging.md).

Fields ARE editable: `PATCH /api/v2/field/{id}` is the only verb the route
allows (`PUT`/`POST` return 405, which reads like "not supported" and is why
this was long assumed UI-only). The endpoint **replaces** `type_config` instead
of merging it. A drop_down sent without its options is rejected (`FIELD_022`),
and one sent with a subset silently deletes the rest along with every task value
pointing at them. `fields update` exists so you never hand-roll that: it reads
the current options, carries each surviving option's `id` through, and sends the
full array. Reach for raw curl only if you really want options destroyed.

**A 200 on that PATCH does not mean the rename landed.** Sending `{name, type_config}`
on the owner's token answered 200, renamed every option and left the field still called
`Taille`; the same call on the team token answered `403 FIELD_262`. So read the name back
with `cu fields list` rather than trusting the status, and when neither token is entitled,
create the English field beside the French one and remove the old one with
`cu fields delete <id> --list <id>`, which goes through the private API because the public
`DELETE /field/{id}` is a 405. Measured 18 Sep 2026 renaming the demo CRM into English.
**And send `type_config` back whole on any field, not only a dropdown**: a currency field
renamed without it answers `400 FIELD_027 / Currency requires a currency_type`.

Running out of dropdown options is the usual cause of a half-empty field. Check
the option list before concluding the values were never filled in.

**A field can be scoped to specific custom task types, and then it is invisible
on every other type.** `--task-types` asks for the field's `applied_objects` and
names the types; a field every type carries prints `all`. A scoped field is
missing from `cu task get <id> --fields` on a task of another type, and
`task field set` answers 400 rather than saying the field does not apply. So a
value that will not stick is a task-type mismatch as often as a bad option id:
read the scope first, and change the type with
`cu task update <id> --custom-item <typeId>` (`cu workspace task-types` lists
them) if that is the real fix.

**`fields list` prints option labels, not option ids, and `task field set` will not
accept a label.** A dropdown takes an option id or its orderindex; a `labels` field
takes option **ids only** (`FIELD_158` on anything else, including an index). Read the
ids straight off the API, then set:

```bash
TOKEN=$(python3 -c "import json;c=json.load(open('$HOME/.config/clickup/config.json'));t=c['tokens'];print(t[0]['token'])")
curl -s -H "Authorization: $TOKEN" "https://api.clickup.com/api/v2/list/<listId>/field" \
  | jq -r '.fields[] | select(.name=="Socials") | .type_config.options[] | "\(.orderindex) \(.id) \(.name // .label)"'
cu task field set <taskId> --field <fieldId> --json-value --value '{"add":["<optId>","<optId>"]}'
```

Watch the orderindex when a dropdown's labels are numbers: options that read
`1 | 2 | 3` have orderindex `0 | 1 | 2`, so `--value 1` sets the option labelled **2**, not 1.

**A `list_relationship` or `dependency` field's value is an array of task IDs**, not a
single ID. Setting it replaces the whole array, so read the current value first when
adding one link rather than dropping the others. A `list_relationship` field also points
at one specific list: two fields with the same name are only true duplicates when
`type_config` names the same list.

**Dates want epoch milliseconds everywhere, including `task create`/`task update`.** A
`--start-date 2026-09-14` is accepted without error and lands on the wrong day (it came
back as start Sep 7 / due Sep 11 on 2026-08-08). Date custom fields reject the string
outright (`FIELD_017`). Compute the epoch first, and pair it with
`--start-date-time false` / `--due-date-time false` (or `--all-day` on a custom field)
for a date-only value:

```bash
MS=$(python3 -c "import datetime;print(int(datetime.datetime(2026,9,14,tzinfo=datetime.timezone.utc).timestamp()*1000))")
cu task update <taskId> --start-date $MS --start-date-time false --due-date $MS --due-date-time false
```

Read the dates back after writing them. That is how the wrong-day bug above surfaced.

### Tags

Tags are space-scoped: a tag must exist in the space before a task there can carry
it, and every tag verb takes the **name**, never an id.

```bash
cu tags --space <id> [--json]                  # list a space's tags
cu tag create --space <id> --name "🗑️ delete" --fg "#ffffff" --bg "#e74c3c"
cu tag update "old name" --space <id> --name "new name" --fg "#ffffff" --bg "#e74c3c"
cu tag delete "tag name" --space <id>          # strips it from every task in the space
cu task tag add <taskId> "🗑️ delete"
cu task tag remove <taskId> "🗑️ delete"
```

`tag update`'s `--fg`/`--bg` carry defaults, so a rename that omits them repaints the
tag black. Pass the existing colours through. A tag write also bumps the task's
`date_updated`, which erases any staleness you were measuring.

Every tag write needs the owner-token override; a member token has no `manage_tags`.

