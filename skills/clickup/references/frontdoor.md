# The private frontdoor: statuses, templates, automations, capture

The public API writes no statuses, templates, automations, dashboards, agents or Workload capacity.
The web app's own frontdoor does, and `cu` wraps it. Commands: `cu statuses`, `cu space statuses`,
`cu list statuses`, `cu template center|save`, `cu dashboards`, `cu agents [get <id>]`,
`cu automations [--space|--folder|--list <id>|get <uuid>|count|catalog|webhooks|webhook-set|apply-template|delete]`.

## Auth

- These use a captured browser session, not `pk_` (the frontdoor rejects it). Run `cu net capture`
  recently: the bearer is a JWT good for about 48 hours, `cu net auth` prints its real `exp`, and
  nothing renews it.
- `CU_TEAM_ID` doesn't pick the workspace here. The session does, so `CU_TEAM_ID=<id> cu fields all`
  silently reads whatever workspace was captured last. Re-capture against the target first.
- Capture the session. Don't reuse `app.clickup_jwts`: those come from a password session with no
  `workspace_id` claim and fail `JWT_015 Token missing workspace_id`.
- Record the call once, script it forever. `cu net capture <url>` learns a surface's reads
  (reloads with the recorder first: 67 requests on the dashboards page against 5 injected live).
  `cu net record [url] --seconds 90` learns a write flow while you click it, and records nothing if
  you don't. Then `net summarize <log> [--grep x]`, `net show <log> --index N`, `net replay`
  (`--url '<host><path>'` probes a guessed endpoint in one second). A replay of POST, PUT, PATCH or
  DELETE needs `--allow-write` and is a real mutation, so read the body with `net show` first.
- Don't guess a path. Eight guesses for the webhook configurations all 404ed. When a click flow needs
  an endpoint and the recorder's tab isn't the one you drive, patch `window.fetch` and
  `XMLHttpRequest.prototype.open` in your own ego page with `page.evaluate` to push `[method, url]`
  into `window.__calls`, click with `page.mouse.click`, then read `window.__calls`. It found the
  endpoint in one click. Then replay it with `cu net replay --url`.
- A workspace your own account doesn't belong to needs ego signed in as an account that does. Then
  `cu net capture` against that workspace and `cu net auth` to confirm the workspace id.
- Neither recorder sees WebSocket traffic, so a missing endpoint may not be HTTP. Vocabulary: a subcategory is a list, a category is a folder and a project is a space. Automation parent-type codes are
  `4` space, `5` folder and `6` list. `/automation/space/{id}` means the WORKSPACE (`ACCESS_700` on a
  space id). There is no public agents route (all 404).

## Statuses

- Write a space's statuses with `cu space statuses <id> --set "to do,in progress,complete:custom,closed"`,
  never `PUT /v2/space`: it answers 200 and keeps the old array. Entries are `name[:type][:#color]`;
  the first defaults to `open`, the last to `closed`, the rest `custom`; a status left out is removed.
- A list with its own lifecycle gets `cu list statuses <id> --set ...` (`--inherit` undoes it), never
  a status dropdown field. It allows one `closed` (a second end state is `done`) and refuses while
  tasks are in a status the new set drops, so move them first.

## Templates

`cu template save <taskId> --replace` and
`cu automations apply-template <listId> --template t-... --name "..."` handle these (verified 29 Sep 2026):

- A save moves the template's `original_id`; an update must send the current one as `template_id`
  or it 404s `ACCESS_031`. The `t-` id stays put and rules point at it. `--replace` looks it up by name.
  A second template with the same name leaves the rule on the old one.
- Save leaves Copy settings for Statuses (`old_statuses`) off, or the template drags list statuses
  wherever it lands. The applied template leaves name, status and type alone.
- The rule is created with template-made and API-made tasks switched off as sources, so it can't fire
  on its own output. `--from-api` turns the API source on, the only way to test from a script: create a
  task, wait about 20 seconds, read it, delete it, then recreate the rule without the flag.
- Replace a rule and never edit it. `PUT /automation/workflow/{uuid}` answers 200 and ignores
  `actions` (name, `active` and `trigger` write), so `apply-template` deletes a rule of that name
  first. `DELETE /automation/workflow/{uuid}` answers 200 empty.
- An AI-agent automation on a list beats a template rule: the agent rewrites name and description
  after the template lands. A list an agent owns gets no template rule.
- Space, folder and list templates have no CLI path: click Templates > Save as template in the
  task menu (labels flip French and English mid-session). The save is asynchronous, so wait for the
  second toast, Template created: the first lies.

## Other rules

Any rule is `cu net replay --allow-write --method POST --url <frontdoor>/automation/subcategory/{listId}/workflow --body '{...}'`
(`/filters/project/{spaceId}`, `/filters/category/{folderId}` for space and folder rules). A
space-level rule doesn't show in the list-level listing, so "no automations" often means one level up.
`AUTO_062` means an unknown trigger name, so probe. Verified 30 Sep 2026:

- Time tracked is trigger `time_spent`, `trigger_on` `TASK|SUBTASK|ALL`.
- Status changed is `status` with `conditions: [{after: [{status_id}]}]`; the `status` action takes
  `{status: {status_id}}`. Ids come from `GET /team/{id}/space`.
- A dropdown picked is trigger type `cf_<fieldId>` with `conditions: [{after: [optionId]}]`.
- Add a subtask is action `create_subtask` with `{name, template_id: "t-..."}`.
- A field is empty is a second condition `{op: "eq", field: "cf_<id>", value: []}` (text) or `""`
  (dropdown). Any other operator is accepted and never fires.
- The API accepts a rule that never fires. Create a task, then wait about 12 seconds and read the status.
- Call webhook: `webhookv2` (saved `webhook_configuration_id`, preferred) or `webhook` (bare
  `endpoint`). The body is `{auto_id, trigger_id, date, payload}`; `payload` has `id`, `custom_type`,
  `subcategory` (the list id), `fields[]` as `{field_id, value}` (a dropdown's value is the
  orderindex), dates as epoch-ms strings, and `users[]` as bare ids with no names. Learn the shape
  from the receiver.
- **The URL isn't in the rule.** A `webhookv2` action stores only `webhook_configuration_id`, and
  `cu automations get` shows no URL. `cu automations webhooks` lists every configuration with its URL,
  and marks `test` on an n8n `/webhook-test/` one. That path only listens while the n8n editor waits
  for a test event, then answers 404 and ClickUp shows no error. Mon Expert RGPD had two rules on it,
  and nothing ran from March to 6 Oct 2026. `cu automations webhook-set <id> --url <prod url>` fixes it.
  The endpoint is `/auto-webhooks-service/v1/workspace/{team}/webhook-configuration[/{id}]`. PATCH
  answers 405, so the write is a PUT that resends url, title, description, payloadType, status and
  customParameters. Header values read back as `********`; leave `customHeaders` out of the body and
  the stored ones stay. Deleting a configuration and creating one are not wrapped, because nobody has
  captured them yet.
- Test a rule end to end by changing the trigger field on one task of a test list, then read the
  receiver's executions. Clearing a dropdown fires the rule again.

## Editing a rule in the UI

Deep link: `https://app.clickup.com/<teamId>/v/o/s/<spaceId>?automation_id=<uuid>&deeplink=automation`.

- The first action has no trash icon (a rule can't drop to zero actions). Retype it, then delete the
  duplicate card. One `+` click can add 2 actions, so count the cards in the footer sentence.
- Cancel sits left of Save a thumb apart; a click meant for Cancel committed an edit. Read back with
  `cu automations get <uuid>`, and never open the builder on a live rule with an edit half made.
- The modal is taller than the window and ignores `scroll`: grow the viewport with
  `cdp('Emulation.setDeviceMetricsOverride', {width: 1512, height: 1500, deviceScaleFactor: 1, mobile: false})`,
  and re-apply after each action pick.
- In the Manage panel's list switcher, clicking the space row scopes to space level, where a list's
  rules are invisible ("Let's create your first Automation"). Press Escape after a pick. Name and
  description edit inline with no Save button; aim at the description's left edge or the click lands
  in Search and the typed text filters the list. To change an action, use the pencil, edit it, then Save.
- For Task created > Apply template by hand: lightning bolt, Manage automations, Add
  Automation, scope Tasks (not "Tasks or subtasks", or it fires on the template's own subtasks),
  action Apply template. The yellow note that the Templates source was switched off is expected.

## Pin a list description

`PUT /list/{id}` needs `markdown_content`, not `content`. The pin is a per-view setting on the list's
required List view `6-{listId}-1`, created lazily, so the first pin on any list is manual: Customize
view > More options > Layout options > Pin description, then Save view. After that,
`PUT /view/6-{id}-1` with the full view body pins and unpins in bulk.
