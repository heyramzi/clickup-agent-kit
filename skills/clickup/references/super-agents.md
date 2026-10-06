# Super Agents and their MCP servers

Connecting a server and building an agent are the longest click paths in this skill and the ones most likely to move under a redesign. Read them here rather than from memory.

## What actually constrains an agent

Four panels, only two of them limit anything. **Instructions** are the real
scope. **Skills** (tools) are the real permission boundary: a tool it lacks it
cannot do, a tool it has it will eventually reach for. **Knowledge** barely
narrows anything (below). **Triggers** decide only when it runs.

**Knowledge cannot really be narrowed.** Workspace Access is forced on: turning
it off answers *Public Workspace knowledge is required for Agents* (verified 21
Aug 2026). **Add from ClickUp** adds emphasis to a space or doc, never a hard
boundary — an agent can always read the whole workspace, so the only real guard
against answering about the wrong client is a sentence in Instructions naming
the space it owns and telling it to ignore the rest.

**Give it the smallest tool set that answers its question.** Read-only on
systems of record, write only on the object it was handed. Money, agreements
and rate cards get read tools and nothing else; an agent that produces work may
comment on and move the task it was assigned, nothing more. Reading is
reversible, writing is not. One agent, one question: twenty tools means it
picks the wrong one more often, and a misfire can't be debugged because you
can't tell which half went wrong.

**Anything long, per-client, or likely to change belongs in a Doc, not
Instructions.** Add it under Knowledge → Add from ClickUp → Docs, and tell the
instructions to name the doc and treat it as the source of truth. Instructions
stay the method; the doc is the facts, edited without touching the agent.

**Give ambiguity somewhere to go.** A verdict field needs a third value
(Unclear, Not in our records yet) and a ban on folding it into the decisive
one, or the agent picks whichever answer makes the story bigger. A soft
judgement usually means an underspecified source: an explicit **excludes** list
on a source of signed deliverables fixed wrong verdicts that harder
re-prompting never did.

**Ban the unsourced number.** Never state a number not read from a tool; if a
tool returns nothing, say so instead of estimating.

**Its description is on screen even when the shot never opens the agent.** The
one-line description shows in the agents list itself, so a demo agent's
description follows the same invented-data rule as everything else on camera.

## Connect an MCP server, and give its tools to a Super Agent

Shipped in release 4.07 on 18 Aug 2026. There is no API for it; the whole surface
is App Center.

From `/<teamId>/settings/apps`, click **App Center**. The row highlights on the
first click and the modal opens on the second; it fades in over about four
seconds, so a click aimed at it during the fade lands on the page behind and
dismisses it. Wait, screenshot, then click.

In the modal's left rail, under **AI**, pick **MCP Servers**. Two doors:

- **The catalogue**, eighteen apps on 20 and 21 Aug 2026 (Amplitude, Atlassian,
  Canny, Canva, Clay, Dropbox, GitHub, Hex, HubSpot, Intercom, Linear, Mixpanel,
  Notion, Sentry, Slack, Stripe, Supabase, ZoomInfo). Each is OAuth into that
  account, so connecting one puts whatever it holds in front of the agent. Never
  complete one on a live account for a recording.
- **Add Custom MCP Server**, which takes any URL.

**Which workspace am I in?** The rail carries a **Custom MCP** category only
where a custom server is connected, so that entry is the fastest proof. The
catalogue also holds ClickUp's own card, **MCP Servers**: its Personal tab with
nothing connected reads "Couldn't load tools", the empty state, not a fault.

The custom path: permissions (**For all members** or **Just for me**) →
**Next** → Name, URL, Description, Authentication Method → **Next**. Auth offers
OAuth, **Authorization header** (a static token, pasted as `Bearer <token>`), or
**No Authentication**; the last two also take Custom Headers. On success ClickUp
calls the server's `tools/list` itself and shows the discovered tools — the proof
the connection works. ClickUp's help page calls the middle option "API key"; the
live UI says **Authorization header** (re-read 21 Aug 2026). Trust the UI.

**Pick the permission scope first, not last.** A personal connection has no
credentials when the person is not logged in, so a scheduled agent using one does
not error, it silently reports only what it could reach. Anything on a schedule
needs the workspace connection.

**Giving an already-connected server's tools to an agent is a different screen,
and it has a trap.** In the agent's Skills panel, **Add tools** opens a modal whose
top right carries **Custom MCP Server**. That button is not a picker: it restarts
the connect-a-new-server flow, and finishing it would stand up a second copy of a
server already connected. Cancel out of it. The connected servers sit further down
the same modal, under a **Custom MCP Servers** heading below the catalogue apps:
scroll to it, click the server, then **Add all** or pick tools one by one.

**That modal paints late** — a click registers and the result appears four to
six seconds later, so a second click aimed at the same place lands on whatever
has since moved under the cursor. Click once, wait, screenshot.

**Searching it returns tool groups, not tools.** Typing `status` returns one row,
**Tasks and subtasks** (8 tools), and adding that group is how an agent gets the
ability to change a status. Individual tool names mostly do not match; search by
the thing you want done. A custom server's own tools do match by name. Give
tools through this form, not through the builder chat.

A **Finish Setup** banner naming the server in Skills as **Unavailable** paints
on some loads and clears on others while the tools still work. It settles
nothing either way: ask the agent the question and read the answer.
