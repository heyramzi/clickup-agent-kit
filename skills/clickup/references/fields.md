# Custom fields, views and tags

Read this before a write you haven't already made once. These entries each cost somebody a run.

## Setting a value

- A dropdown takes an option index or uuid, never the label (`400 FIELD_011`). A `labels` field
  takes option ids only (`FIELD_158` on anything else). `cu fields list` prints labels, not ids, so
  read them off `GET /api/v2/list/{id}/field` (`type_config.options[]`), then
  `cu task field set <id> --field <fieldId> --value <optionId>`. A labels field adds with
  `--json-value --value '{"add":["<optId>"]}'`.
- Labels that are numbers mislead: options `1 | 2 | 3` have orderindex `0 | 1 | 2`, so `--value 1`
  sets the option labelled 2. Reads come back as the orderindex too, and orderindex 0 is falsy, so
  `value ? x : null` treats it as empty.
- A `list_relationship` or `dependency` value is an array of task ids and a set replaces the whole
  array, so read before you add. 2 relationship fields with one name are duplicates only when
  `type_config` names the same list.
- A field scoped to specific task types is invisible on other types, and `task field set` answers
  400 without saying why. `fields list --task-types` shows the scope; fix with
  `cu task update <id> --custom-item <typeId>` (`cu workspace task-types`).
- A dropdown with half its values empty usually ran out of options. Check the option list first.

## Creating and changing fields

- `cu fields create` on a list makes a list-only field. `POST /v2/space/{spaceId}/field`
  (undocumented, same body shape, takes `drop_down` and `currency` config) creates it once for the
  space, which suits one automation that must address several lists.
- `POST /list/{id}/field` returns the EXISTING field when the name and type already exist, so two
  "creates" can be one field.
- Field edits are `PATCH /api/v2/field/{id}` only (PUT and POST 405, so it looked
  UI-only). It replaces `type_config`: a dropdown sent without its options is rejected
  (`FIELD_022`) and one sent with a subset deletes the rest and every task value on them. A currency
  field renamed without `type_config` is `400 FIELD_027`. `cu fields update` reads the current
  options and sends them all back. Use that (`--add-options`, `--rename-options "old=new"`).
- A 200 on that PATCH isn't a rename: the owner token answered 200, renamed the options and left the
  field name alone (18 Sep 2026), and the team token answered `403 FIELD_262`. Read the name back.
  When neither token is entitled, create the new field beside the old one and
  `cu fields delete <id> --list <id>` (public `DELETE /field/{id}` is a 405).

## Merging and deleting duplicates

`cu fields all`, `duplicates [--loose]`, `merge --into <keep> --from <id,id>`, `delete` use the
private API, so `cu net capture` first. A merge moves every task value server-side and can't be
undone: run `--dry-run`, show the groups, survivor and `dropped` count, and get a human yes.

- The ranking `duplicates` prints is a default. Override it when an older field is the one
  automations point at. Rename after merging, never before.
- Per merged-away field the dry run shows `values / moving / dropped`. If both fields have a
  value, the kept one wins silently. A group whose whole count is `dropped` was filled in parallel,
  and merging loses a column of data.
- The survivor is rebuilt: its id and every option id change, and a `new_drop_down` comes back in
  the legacy shape (values read as orderindex). Extras on `type_config`, such as an AI field's prompt, are lost. Save `cu fields all --json` before merging and restore by hand.
- Grep the old field id across views (`cf_<id>` columns, filters), automations
  (`cu automations --list`), integrations and dashboards before merging, then repoint.
- The dry-run search misses archived tasks, so a merge can move a value the count never showed.

| Refusal | Meaning |
| --- | --- |
| `Field types do not match.` | Only same-type merges exist. Recreate and re-enter by hand. |
| `Not all source field options are specified.` | Map each option: `--map "old=new"` or `--add-missing`. |
| `Not all custom fields were found.` | Usually the same field twice (see create above). |
| `FIELD_192 Parent must be included` | `fields delete` needs `--list`, `--folder` or `--space`. |
| `FIELD_209 Cannot remove a field directly from the workspace level` | Re-home with `PUT /customFields/v2/field/{id}` and `project_id`, then delete there. |
| `FIELD_262 Access denied for updating field api` | A member token. Use the owner profile. |
| `FIELD_214 Field already exists in parent` | Already applied there. Nothing to do. |

## Views

- System views (`4-SPACEID-28`, `5-FOLDERID-28`, `6-LISTID-8`) can't be deleted; user-created ones
  look like `<workspaceCode>-XXXXX`. `cu view delete` prompts `Delete <id>? (y/N)`: without a `yes |` it
  leaves the view alive while the script goes on, and one run ended with 2 views of one name.
- `--columns-json` must not list `name`. Name always renders, so listing it adds a duplicate
  header and every later header shifts one column, which reads as fields with no values. A custom
  field column needs `cf_<fieldId>` (bare id 404s `View Not Found`); built-ins (`assignee`,
  `status`, `priority`, `dueDate`) take no prefix.
- A `list`, `board` or `table` view with `grouping.field: none` is one flat column. Group on
  `status` or a field. `PUT /view/{id}` replaces: echo back `name, type, grouping, divide,
  sorting, filters, columns, settings` or the rest resets.
- A filter entry is `{"field":"assignee","op":"ANY","determinor":null,"idx":0,"values":[<int ids>]}`.
  String values are dropped silently. `GET /view/{id}/task` ignores all but one AND'ed field and
  paginates 30 per page (loop to `last_page: true`). v3 view endpoints 404.
- A ClickApp that's off eats values silently. A priority stores and reads back on a space with
  Priorities off, so a view grouped on it puts all tasks under "No Priority". Same for Sprints,
  Milestones and native points. Check `GET /space/{id}` `features`: a missing key means off. Turn it on
  with `PUT /space/{id} {"features":{"priorities":{"enabled":true}}}` and stored values appear.

## Tags

Tags are space-scoped, must exist before a task carries one, and every verb takes the name.
`cu tag update` carries default `--fg`/`--bg`, so a rename without them repaints the tag black. A
tag write bumps the task's `date_updated`, which erases any staleness you were measuring. Tag
writes need the owner token.
