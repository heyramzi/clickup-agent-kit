# Merging custom fields

`cu fields merge` moves every task value from the merged-away fields onto the kept
one, server-side, and cannot be undone. Price it with `--dry-run` first and get a
human yes before running it for real: showing the groups, the survivor and the
dropped count is the only gate this command has.

```bash
cu fields duplicates [--loose]                      # groups near-twins, survivor first,
                                                     # ranked by workspace level, then
                                                     # locations applied, then option
                                                     # count, then age. --loose also
                                                     # pairs singular with plural.
cu fields merge --into <keepId> --from <id,id> --dry-run
cu fields merge --into <keepId> --from <id,id> --dry-run --add-missing
```

The ranking `duplicates` prints is a default, not a decision: override it when an
older field is the one automations and integrations already point at, or the
better-named field is the smaller one. Rename after merging, never before.

`--dry-run` prints the option mapping and, per merged-away field, `values` /
`moving` / `dropped`. `dropped` is how many task values die: where both fields
carry a value, the kept one wins. A group whose whole value count is `dropped` is
one where the two fields were filled in parallel, and merging it loses a column of
data, not just a duplicate label. `--add-missing` adds an option the survivor is
short; on a dry run it only prints `(added to <field>)` and writes nothing.

## What the merge does, exactly

Measured on a live workspace, 26 Aug 2026:

- **Values move.** A task that carried a value only on a merged-away field carries
  it on the survivor afterwards, whatever list it lives in.
- **The kept value wins a collision**, silently. Nothing reports it after the
  fact; the dry run is the only warning.
- **Locations union.** Two list-level fields on different lists become one field
  applied to both lists.
- **The survivor is rebuilt.** Its field ID and every option ID change, its
  `date_created` resets, and a `new_drop_down` dropdown comes back in the legacy
  shape, so its task values then read back as an option **orderindex** instead of
  an option ID.
- **Extras on `type_config` do not survive.** An AI-filled dropdown loses its
  prompt. Read the old field's `type_config` from the pre-merge inventory
  (`cu fields all --json`) and put it back by hand.

## Repointing after a merge

The new field ID breaks anything holding the old one. Grep the old ID across the
workspace before merging, so the list of places to repair exists before the ID is
gone, then check:

- **Views.** A custom field column is `cf_<fieldId>` and a filter holds the field
  ID. `cu views list` then `cu view get <id>`.
- **Automations.** `cu automations --list <listId>` per list the field touches.
- **Integrations.** Make scenarios, n8n workflows, webhooks, and any script in
  this repo.
- **Dashboards** and saved filters.

## Refusals, and what each one means

| Message | Meaning |
| --- | --- |
| `Field types do not match.` | Only same-type merges exist. Different types means recreate and re-enter by hand. |
| `Not all source field options are specified.` | Every option of the merged-away field needs a target. `--map "old=new"` points one anywhere, `--add-missing` adds it to the survivor. |
| `Not all custom fields were found.` | Usually the same field twice: `POST /list/{id}/field` returns the EXISTING field when the name and type already exist, so two "creates" can be one field. |
| `FIELD_192 Parent must be included` | `fields delete` needs `--list`, `--folder` or `--space`. |
| `FIELD_209 Cannot remove a field directly from the workspace level` | Re-home it first (`PUT /customFields/v2/field/{id}` with `project_id`), then delete it from there. |
| `FIELD_262 Access denied for updating field api` | The public PATCH refuses a member token. `cu fields merge --add-missing` goes through the session instead; for `cu fields update`, switch to the owner profile. |
| `FIELD_214 Field already exists in parent` | The field is already applied there. Nothing to do. |

## Two things the numbers do not cover

The filtered task search behind the dry run does not see **archived** tasks, so a
merge can move a value the count never showed. And a `list_relationship` field
points at a specific list: two of them with the same name are only duplicates
when they point at the same list. Read `type_config` before grouping those.
