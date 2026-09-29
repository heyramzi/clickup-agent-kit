# Task templates, Task-created automations, and the pinned description

The public API writes none of these. The web app's own frontdoor writes two of them, and `cu` wraps that, so start there:

    cu template save <taskId> --replace        # save, or re-save over the same t- id
    cu automations apply-template <listId> --template t-… --name "Member created → Apply the member template"

Both read the workspace off the `cu net capture` session, so capture the target workspace first. The traps they already handle, verified 29 Sep 2026:

- Every save moves the template's `original_id`, and an update must send the current one as `template_id` or it 404s `ACCESS_031`. The `t-` id stays put, and a rule points at that one. `--replace` looks up the current `original_id` by name.
- The save body is the dialog's checkboxes (`old_due_date`, `old_statuses`, `custom_type` and so on), stored as the template's `options`. `old_statuses: false` is the one that matters: it's "Copy settings for Statuses", which would drag list statuses into wherever the template lands.
- The rule is created with template-made and API-made tasks switched off as sources, so it can't fire on its own output and a builder's inserts keep their content. `--from-api` turns the API source back on, which is also the only way to test a rule from a script: create a task, wait ~20s, read the description, delete it, then recreate the rule without the flag.
- A rule is replaced, never edited: `apply-template` deletes a rule of the same name first, because PUT ignores `actions` (below). `DELETE /automation/workflow/{uuid}` answers 200 with an empty body.
- The applied template leaves the task's name, status and type alone, even on a list with its own statuses.

A demo builder can run all of it from a content file.
The click paths below are the fallback, and still the only route for space, folder and list templates and every other kind of rule. They were walked on the live workspace; the labels are given in both languages because the UI flips between them mid-session, and every coordinate is a hint rather than a target.

## Save a task as a workspace task template

From `https://app.clickup.com/t/<TASK_ID>`, waited and loaded:

1. `...` in the task header top right, about (1329, 53) at 1428 wide. The Share
   button sits immediately left of it, so confirm on the screenshot that a menu
   opened with Favorite / Move to / Templates. If Share opened, close it without
   submitting and click further right.
2. **Templates** / **Modèles**, about (1187, 352).
3. **Save as template** / **Enregistrer en tant que modèle**, the second item in
   the submenu, about (1171, 533).
4. The properties dropdown reads **21 of 22 properties**. Open it and uncheck
   **Copy settings for Statuses** / **Copier les paramètres des statuts** so it
   reads 20. That option copies list status settings into wherever the template
   lands, which is not what a task template is for. Close the dropdown by
   clicking the **Template contents** label.
5. Name it, then **Save Template**.
6. ClickUp shows an interim "your template is being created" toast, then the real
   one: **Template created 🎉**. The save is asynchronous. Wait for the second
   toast, because the first one lies about being finished.

The dialog remembers the property choice within a session but resets to 21 on a
fresh page, so re-check it every time.

## Update a saved template in place

A template that is edited in place keeps its id, so every automation pointing at
it picks up the new content on its next run. Creating a second template with the
same name does not, and leaves the automation on the old one.

Same path as above, except step 3 is **Update existing template** / **Mettre à
jour le modèle existant**, the THIRD submenu item, about (1192, 556). Then pick
the template by name from the searchable picker. Templates made by these runs
show as "by <your workspace name>" with a recent timestamp, which is what tells them apart from
an older one with a similar name.

Then: edit the source task first, re-save over the template, and the loop is
closed. Content lives in its `.mjs` source, the builder writes the task, this
path publishes it.

## Task created → Apply template

1. Open the list, click the **lightning bolt** in the top right header row, about
   (1277, 53). A panel titled **AI Fields** opens.
2. **Manage automations** at the bottom of that panel, then **Add Automation**.
   The empty state has its own Add Automation button in the middle.
3. Trigger scope defaults to **Tasks or subtasks**. Change it to **Tasks**.
   Without this the rule fires on the subtasks the template itself creates.
4. Action defaults to Update status. Open it, type `template` (or `modèle`), pick
   **Apply template**.
5. A yellow note appears: the **Templates & duplicated tasks** trigger source has
   been turned off for this action, and the source count drops from 10 to 9.
   ClickUp does that on its own so an import cannot retrigger the rule. Leave it.
6. **Select a template** opens the Template Center. Search, then click the card
   whose title matches exactly, then **Use Template**.
7. Name the rule, add its one-line description, **Create**.

## What the automation API does and does not let you do

Three private endpoints, verified 1 Sep 2026 on a live space:

    POST /automation/filters/project/{spaceId}/workflow    # space-level rules
    POST /automation/filters/category/{folderId}/workflow  # folder-level
    POST /automation/filters/subcategory/{listId}/workflow  # list-level
    GET  /automation/workflow/{uuid}                        # one rule in full

They are `cu automations --space|--folder|--list` and `cu automations get <uuid>`. The
vocabulary trips you twice: a space is a **project** here, and `/automation/space/{id}`
means the WORKSPACE, answering `ACCESS_700` when handed a space id. **A space-level rule
does not appear in the list-level listing**, so a rule you know exists reading as "no
automations" usually means it is defined a level up.

**`PUT /automation/workflow/{uuid}` answers 200 and silently ignores `actions`.** Name,
`active` and `trigger` write; the action array comes back exactly as it was, with a fresh
`last_updated` to make it look like something happened. Confirmed three ways: bare action
object, action with a client-generated uuid, and the full workflow object round-tripped.
So **every action edit is a delete and a create** (`POST /automation/subcategory/{listId}/workflow` takes the full rule; `cu automations apply-template` is the one kind wrapped) or a click path, and reading the rule first is still worth it: the
action ids, a `webhook_configuration_id` and the trigger's source flags are all invisible
in the UI.

## Editing the actions of an existing rule

Open the rule directly with its own deep link rather than hunting it in the panel:
`https://app.clickup.com/<teamId>/v/o/s/<spaceId>?automation_id=<uuid>&deeplink=automation`.

- **The first action has no trash icon.** ClickUp will not let a rule drop to zero actions,
  so the delete only appears from the second card down. To remove the first one, retype it:
  open its own dropdown, pick the replacement, then delete whichever card became the
  duplicate. Dragging works too and is far more fragile.
- **Save and Cancel sit a thumb apart** at the bottom right, Cancel left of Save. At 1512
  wide they were (1260, y) and (1331, y): a click meant for Cancel landed on Save and
  committed an edit that was only meant to be inspected. Read back with
  `cu automations get <uuid>` rather than trusting which one you hit, and never open the
  builder on a live rule with an edit half made.
- **The modal is taller than the window**, and its inner column does not answer `scroll`.
  Grow the viewport instead: `cdp('Emulation.setDeviceMetricsOverride', {width: 1512,
  height: 1500, deviceScaleFactor: 1, mobile: false})` shows all the actions at once.
  Picking an action from a dropdown resets the override, so re-apply it after each pick.
- **One `+` click can add two actions.** Count the cards in the footer sentence ("When X
  then A and B and C") before assuming what you have.

## Call webhook: two actions, and the payload it sends

The action picker offers **Call webhook** (`webhookv2`, takes a saved
`webhook_configuration_id`) and **Call webhook (Legacy)** (`webhook`, takes a bare
`endpoint` URL). Prefer v2: the configuration is reusable and named, and the picker can
create one inline.

The body it POSTs (axios user-agent) is:

    {auto_id, trigger_id, date, payload: { ...the task... }}

`payload` carries `id`, `custom_type`, `subcategory` (the list id), `fields[]`, and dates
under `time_mgmt.start_date` / `time_mgmt.due_date` as epoch-ms **strings**. Each entry in
`fields[]` is `{field_id, value}`, and a dropdown's `value` is the option **orderindex**,
never the label or the option id, matching how `cu task field set` takes it. It carries
`users[]` as bare user IDs and **no usernames**, so anything that needs a person's name has
to re-read the task. Learn the shape from the receiver rather than from here: with Make,
that is `make hook-logs <hookId> --log <logId>`.

### Editing rules without reloading anything

The Manage panel has a list switcher at its top left with its own search box.
Switching lists there is much faster than navigating. Two traps:

- Results show the space row and the list row indented under it. Clicking the
  space silently scopes the panel to space level, where the list's rules are
  invisible and the panel reads "Let's create your first Automation". If a list
  you know has a rule looks empty, this is why.
- The dropdown stays open after the pick. Press Escape before clicking anything
  behind it, or the click lands back in the dropdown.

The rule's **name and description are inline editable** in the Manage list. Click
the title text, cmd+a, type. Click the grey "Enter description..." line, type.
Click empty space below to commit. Both save without a Save button.

Click the description line at its left edge. A click too far right or low lands
in the panel's Search box instead, where the typed sentence silently becomes a
filter and the list empties. Clear it with the x if that happens; nothing was
written to the rule.

Changing a rule's **action** needs the full form: hover the row, click the pencil,
edit, **Save**.

Deleting a rule: hover the row, red trash icon, then Delete in the confirm.

## Pin a list description

Not in the list settings menu, and not in List Info. It is a per view setting:
the **⚙ Customize view** in the view toolbar → **More options** → **Layout
options** → **Pin description**. The pin therefore has to be set on whichever
view somebody actually lands on, which for a list is `6-<listId>-1`.
