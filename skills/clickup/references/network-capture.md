# Network capture and the UI's own API

Reference for the `clickup` skill's browser mode. Both halves are about the same thing: reading what the
ClickUp web app calls, then re-issuing it from a terminal instead of clicking it again.

## Record the call once, script it forever

**When a flow is not yet a command, capture it rather than re-clicking it.** Two
modes, and picking the wrong one wastes the run:

- **`cu net capture <url>`** learns a surface's **read** endpoints. It registers
  the recorder before any page script and reloads, so it sees the whole boot
  sequence. On the dashboards page this caught 67 requests where injecting into
  the live page caught 5.
- **`cu net record [url] --seconds 90`** learns a **write** flow. It injects into
  the live page and records while you click, so the flow has to be driven by hand
  during the window. `net drain` empties a long session in stages.

Then read what you caught and re-issue it:

    cu net summarize <log>               # distinct endpoints, ids collapsed to :id
    cu net summarize <log> --grep automation
    cu net show <log> --index 7          # full headers and both bodies
    cu net replay <log> --index 7        # GET, straight from the terminal
    cu net replay --url '<host><path>'   # probe a guessed endpoint

**Probing by replay beats guessing in the browser.** `net replay --url` is a
one-second GET, so a candidate path is cheap to test. That is how
`/templates/v1/team/{id}/templates` was found after the browser route stalled.

`net auth` stores the header set ClickUp's app sends: `Authorization` bearer,
`X-CSRF`, `sessionId`, `X-Workspace-ID`, `build-version`. Replay sends all five.

**A replayed write is a real mutation on a live workspace.** `net replay` blocks
POST, PUT, PATCH and DELETE unless `--allow-write` is passed, so read the body
with `net show` before you send one.

### What the paths tell you

ClickUp's internal vocabulary leaks through, and knowing it saves a wrong guess:
**subcategory = list**, **category = folder**. Automation parent-type codes follow
the same scheme: `4` = space, `5` = folder, `6` = list, as in
`/automation/team/{team}/parentType/6/parentId/{listId}/suggestAutomations`.

Two limits. `net record`'s page-side patch survives in-app navigation but dies on
a full page load, which is why it navigates first and injects second (`net
capture` has no such limit). And neither sees **WebSocket** traffic; ClickUp
pushes live updates over one, so an endpoint missing from a capture may not be an
HTTP call at all.

## Getting off the clicks: what the UI calls

These are click paths because the public API has none. The app does. Call
`read_network_requests` with `clear: true` right before an action and read right
after: the gateway is cross-origin, so every call leaves an OPTIONS preflight
carrying its operation name. Auth is the session JWT (`clickup_jwt_capture`).

Observed 21 Aug 2026 on EU workspaces: App Center and Super Agents share one
GraphQL gateway, `frontdoor-search.clickup-eu.com/graphql/gateway?q=<Operation>`,
where connecting a custom MCP server is `ConnectAppMutation` and an open agent
page polls `ListActiveAgentRuns`. Everything else is REST on a per-region host,
`frontdoor-prod-eu-west-1-3.clickup.com/<service>/<version>/...` (`hierarchy/v3`,
`comment-service/v3`, `scheduling/v1`). The Super Agent Builder streams, so its
writes never appear as XHR at all.

**The agents service is REST, and the CAPTURED session reaches it.** Corrected
24 Aug 2026. An agent is a view object, so its id has the doc-page shape
(`kg1h7-53895`) and its page is `/<teamId>/ai/agents/<agentViewId>`. Both routes
are 200 from a script and both are wrapped:

    cu agents              # every agent: id, name, active, updated
    cu agents get <id>     # the whole agent_config

This section previously recorded the opposite, that `automation/*` was
browser-only because the call returned
`{"err":"Token missing workspace_id","ECODE":"JWT_015"}`. That finding was real
but it was about the wrong token. The JWTs in `app.clickup_jwts` come from a
password session and carry `{user, ws_key, session_token}` with no
`workspace_id` claim, and no header supplies it. The token the web app itself
sends to the frontdoor is a different one, and `cu net capture` stores it along
with the `X-Workspace-ID` header that goes beside it. With that pair, every route
under `automation/` answers: agents, the automation catalog, per-list workflows
and their counts.

**So the rule is: capture the session, do not reuse `app.clickup_jwts`.** Those
two token sources are not interchangeable, and the 400 is what tells you the
wrong one is in play.

Do not go looking for a public endpoint either. Verified 21 Aug 2026 against
`developer.clickup.com/llms.txt` and the live API: there is no agents route on
v2 or v3 (`/v3/workspaces/<id>/agents`, `/v2/team/<id>/agent`,
`/v3/workspaces/<id>/ai/agents` all 404). The only agent-shaped thing in the docs
is ClickUp's own MCP server, which points the other way.

Add an operation each time you see one. Once a shape is proven twice, it belongs
in the `cu` CLI, not here.
