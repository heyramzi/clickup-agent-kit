---
name: clickup
description: "Runs ClickUp: the cu CLI for tasks, docs and hierarchy (default), scripted bulk work (cleanup, demo enrichment, doc-name standardizing), and the browser for what has no API (templates, automations, dashboards, statuses, Workload). Use for any ClickUp task, doc, bulk-update or browser-only job."
tags: [drives, clickup, browser]
lane: general
---

# ClickUp

The **cu CLI**, **bulk data work**, and the **browser** get you in, in that order. Reach for the
cu CLI first, for anything interactive: tasks, docs, hierarchy, time tracking, and the
private-frontdoor surfaces it now wraps. Bulk data work is scripted cleanup, demo enrichment and
doc-page renaming over curl/Python instead of the CLI. The browser is the last resort, only for
the handful of surfaces neither of the other two reaches yet.

## The kit, and the one map it shares

This skill is the base of a small free kit. Every piece reads `.tasks/config.json` first: the list
id, the real status names and the GitHub repo. Shape, setup and rules:
[`references/tasks-config.md`](references/tasks-config.md).

| Piece | Kind | Job |
| --- | --- | --- |
| `clickup` | skill | Drive ClickUp: the `cu` CLI, bulk work, the browser |
| `board` | skill | Write the brief, start, ship and move a task, from idea to merged PR |

Planning the week in batches with points, and turning a call into tasks, come with Agency Master.

## cu CLI (default)

One global binary, no `clickup` alias:

```bash
cu <command> [args]
```
Install with `npm i -g heyramzi/clickup-agent-kit`, then `cu init`. Public repo, no token to ask
anybody for. Course: **ClickUp Foundations** at
[skool.com/ai-agency-systems-3191](https://www.skool.com/ai-agency-systems-3191). The commands
below all assume `cu` is on PATH.

**Config**: `~/.config/clickup/config.json`, multiple tokens with priority-based fallback.
Override with `CU_API_TOKEN` / `CU_TEAM_ID`. Fallback is per-call token selection, not retry:
every command runs on `tokens[0]` and stops, so a `401 ACCESS_081` is not proof the op is
forbidden, just that the other token owns the object. No `--token` flag; switch for one call:

```bash
export CU_API_TOKEN=$(python3 -c "import json;d=json.load(open('$HOME/.config/clickup/config.json'));print([t['token'] for t in d['tokens'] if t['name']=='owner'][0])")
```

A member token is missing whole scopes, not single objects: `manage_tags` is one, so every tag
write needs the override above. `cu tag create` 401s `ACCESS_081`, `cu tag delete` 401s
`ACCESS_016` (`invalid_permissions: ["delete_tags"]`): different codes for the same missing
scope. Run `cu profiles` / `cu status` for the real write-scope profile name; it isn't always
called `owner`. `cu init` (interactive setup), `cu status` (auth + priority).

Command reference (every command, flags, return shape):
[`references/command-reference.md`](references/command-reference.md): the binary wins when it
disagrees with the table. Views and fields: [`references/command-reference-fields.md`](references/command-reference-fields.md).
Tags, docs, the private frontdoor: [`references/command-reference-advanced.md`](references/command-reference-advanced.md).

**Where the public API stops.** Templates, automations, agents, dashboards and statuses have no
public endpoint but are still `cu` commands, on ClickUp's private frontdoor API:

    cu template center     cu dashboards     cu statuses
    cu space statuses <spaceId> --set "to do,in progress,complete:custom,closed"
    cu list statuses <listId> --set "enrolled,active member,left:done,deceased:closed"   (--inherit undoes it)
    cu agents               cu agents get <id>
    cu automations --list <id>   cu automations count <listId>   cu automations catalog
    cu template save <taskId> --replace      cu automations apply-template <listId> --template t-… --name …
    cu fields all            cu fields duplicates   cu fields merge   cu fields delete

These auth with a captured browser session, not `pk_`, which the frontdoor rejects: run
`cu net capture` recently (the bearer is a ~48h JWT). `CU_TEAM_ID` picks the workspace for the
config but the frontdoor commands read the workspace off the captured session instead:
`CU_TEAM_ID=<id> cu fields all` silently reads whatever workspace was last captured. Re-run
`cu net capture` against the target workspace before trusting `CU_TEAM_ID` on any of these.
A space's statuses are written with `cu space statuses --set`, never `PUT /v2/space`: that
route answers 200 and drops the array, keeping the old statuses with no error. A list that
needs its own lifecycle gets `cu list statuses --set`, never a status dropdown field: it allows
one `closed` (a second end state is `done`), and it refuses while tasks sit in a status the new
set drops, so move them first.

**Role changes below Enterprise**: `cu user update <userId> --admin[=false]` falls through to
the frontdoor (`PUT /team/v1/team/{teamId}`, roles **1 owner, 2 admin, 3 member, 4 guest**)
because the public route 403s below Enterprise. The session must be an owner/admin already: a
member gets `401 ACCESS_009`. Set `CLICKUP_FRONTDOOR_JWT` to an owner's token when the browser
session is signed in as a plain member.

**Output**: interactive gets colour, a pipe or `--json` gets JSON, `NO_COLOR=1` drops ANSI.
Always `--json` when parsing, but: `task get --json` drops custom fields (`task get --fields`
prints them resolved; `--markdown` dumps the description alone); output is JSON then a
`help[...]` block, so parse with `json.JSONDecoder().raw_decode(text)[0]` or
`json.loads(out.splitlines()[0])`; `cu tasks --json` returns `{count, items}` and hides Closed
tasks unless `--closed` is passed.

**Chat**: `cu chat messages <channelId>` truncates to 120 chars with no author: `--full` prints
each message whole plus a `user` id column, `cu chat members` maps ids to names (same for
`chat replies`). One page is 100 messages. A DM 404s on a token that isn't a party to it: run
it on the owner profile.

**`description` stores plain text, `markdown_content` renders markdown**: same endpoints
(`POST .../task`, `PUT /task/{id}`), both return 200 either way, so a `**bold**` body posted as
`description` shows literal asterisks on screen with no error. Use `markdown_content` the moment
the body has a heading, list or table. Writing a Doc page (table rendering, `--mode replace`
destroying embeds): [`references/doc-authoring.md`](references/doc-authoring.md). The writes
that fail on the first honest attempt: dropdowns, `--columns-json`, space-wide fields, field
merging, guest assignment, AI-agent assignee ids:
[`references/writes-that-fail.md`](references/writes-that-fail.md).

## Bulk data work

Scripted workspace management (cleanup, enrichment, bulk ops) over curl/Python directly
against the REST API, not `cu` (which is for interactive single-object use).

Tokens live in `.env.local`, one per workspace (`CLICKUP_API_TOKEN_OWNER`,
`CLICKUP_API_TOKEN_TEAM`): the token picks the workspace, and using the wrong one 401s on
objects that plainly exist: reads as a permission bug, is actually the wrong token.

Request patterns, the hierarchy/task/view CRUD endpoint map, and the columns-update format:
[`references/api-reference.md`](references/api-reference.md). Demo workspace list ids:
[`references/demo-workspace.md`](references/demo-workspace.md).

**Deletable views**: only user-created (`8cbypq9-XXXXX`-style) ids; system views
(`4-SPACEID-28`, `5-FOLDERID-28`, `6-LISTID-8`) error on DELETE: skip them.

**List description pin**: `PUT /list/{id}` needs `markdown_content`, not `content` (both read
back as flattened `content`, so a plain-looking readback isn't proof it failed). The pin lives
at `settings.is_description_pinned` on the list's required List view (`6-{listId}-1`), which is
created lazily: no API call creates it. First pin on any list is one manual toggle: open the
list, click the description icon in the breadcrumb, flip **Pin this description for added
visibility**, close, **Save view**. After that, `PUT /view/6-{id}-1` with the full view body
pins/unpins in bulk.

**Markdown**: ClickUp treats a single newline as a hard break everywhere (lists, tasks, docs).
Keep every paragraph on one source line, or reflow on the way out (join continuation lines,
leave blank lines/headings/`---`/list markers/blockquotes alone). Headings, bold, italics,
bullets, inline code, blockquotes, hr all render; tables don't via `description`. The public
API writes no statuses; `cu space statuses` and `cu list statuses` do. The v3 docs endpoints 500 intermittently:
retry 5xx two or three times with backoff rather than letting one page kill a build.

Demo workspace cleanup workflow and the data-quality bar (task-name formats by list type,
description/date/status standards):
[`references/demo-cleanup.md`](references/demo-cleanup.md). Doc-page naming standardization
(`Company - Type - MM/DD/YYYY`, keyword detection, execution flow):
[`references/doc-naming.md`](references/doc-naming.md).

## Browser only

Last resort. Most of what used to need a click is now a `cu` command (frontdoor list above) or
a recorded/replayed network call: reach for a click path only when neither exists, because it
breaks on a redesign and an API call doesn't. Announce at start: "I'm using the clickup skill,
browser mode."

**Record the call once, script it forever.** `cu net capture <url>` learns read endpoints,
`cu net record` learns a write flow, `cu net replay` re-issues either (writes need
`--allow-write`). Modes, vocabulary, headers: [`references/network-capture.md`](references/network-capture.md).

**`ego-browser` is the route, Claude in Chrome is the fallback**: one heredoc carries a
navigate, a wait, three clicks and a capture in one round trip. The five first-run traps
(`wait()` is seconds, `click` takes one array, `help()` lies, `drainEvents()` drops network
events, the body is an ES module) and the fallback picker:
[`references/driving-the-browser.md`](references/driving-the-browser.md). A white-labelled
workspace redirects and renders twice: navigate, wait 6-10s, screenshot, then click. UI
language flips French/English mid-session; never key a click off a label you didn't just read.
The viewport also resizes itself between loads.

**Demos are invented, always, in their own list.** Never a real client's board with the logo
swapped: even temporarily, because the intermediate state ends up in a take. The blast radius is the whole
workspace on screen (sidebar, agent list, search, breadcrumb), not just the list in frame.

Task templates and Task created rules from the CLI, the `original_id` trap, and the click paths
for everything else (both label languages, a rule's first action has no delete icon):
[`references/templates-and-automations.md`](references/templates-and-automations.md).
Connecting an MCP server, and what actually constrains a Super Agent:
[`references/super-agents.md`](references/super-agents.md). Building one from scratch, and the
Builder chat, which rewrites the prompt only, never the Knowledge section (`workspaceKnowledge`),
set by hand in the profile: [`references/super-agent-builder.md`](references/super-agent-builder.md).

**Two lazy-render traps**: an agent image posted via `![alt](url)` in a comment renders fine but
`GET .../comment` shows `"text": ""` (the image is a sibling `{"type":"image",...}` block) and
the Activity panel lazy-loads it off-screen. A freshly opened list view paints headers first,
custom-field cells a beat later: scroll and back before capturing either.

**Verify with the API, never the screenshot**: a screenshot proves a dialog closed, not that
an automation fired: `cu task create`, wait ~20s, `cu task get <id> --fields`,
`cu tasks --list <id> --subtasks`, `cu task delete <id>` to clean up. Export `CU_TEAM_ID` for
the workspace under test or the CLI points at your default (production).

**Collisions**: an AI-agent automation on the same list beats a template rule (the agent
rewrites name/description after the template lands): a list owned by an agent gets no template
automation. A concurrent session renaming a source task mid-run orphans a saved template's
pointer; re-read from the CLI before repointing, then save a new template and swap it into the
rule.

Screenshots: the `captureScreenshot` helper is 1x and ignores `deviceScaleFactor`; anything
going on camera goes over CDP instead. Crop rather than hide, to keep a real space name out of
frame: [`references/screenshots.md`](references/screenshots.md).

**Subagents parallelize badly here**: one tab per agent, closing the window kills the whole tab
group at once. Forbid `select_browser`/`switch_browser`/`tabs_create_mcp`, tell each agent to
report rather than improvise when its tab dies, and expect to finish the tail yourself. An agent
also reports success when a click landed but the confirmation dialog was missed: re-read the
object before redoing anything.

**Workload**: open `https://app.clickup.com/<teamId>/v/wl/<viewId>` (id from
`cu views list --space <id>`). Two menus on the view bar: what's summed
(`Tasks | Time Estimates | Time Estimates %`, plus any number or dropdown field the space has)
and what the cell shows
(`Daily Scheduled | Daily Availability | Weekly Capacity | Weekly Availability`: Weekly
Capacity is the percentage-and-red-past-100% mode). A dropdown field is summed by its option
label, so a space's own points-style dropdown drives the bar, settable
from the CLI (`cu task field set <id> --field <fieldId> --value <optionId>`). A person's row only
exists once they have scheduled work inside the window. Setting capacity, collapsing rows, the
click paths and selectors: [`references/workload-view.md`](references/workload-view.md).

Dashboards and cards are UI-only and not mapped yet: map the click path the next time one comes
up rather than guessing it now.
