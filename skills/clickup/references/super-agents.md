# Super Agents and their MCP servers

The longest click paths in this skill, and the likeliest to move under a redesign. Verify against
the screen. `cu agents` and `cu agents get <id>` (the whole `agent_config`) read agents over the
frontdoor; an agent's id is a view id (`kg1h7-53895`) and its page is `/<teamId>/ai/agents/<agentViewId>`.

## Constraints

- Instructions are the real scope and Skills (tools) are the real permission boundary: a tool it
  lacks it can't use, and a tool it has it will eventually use. Knowledge barely narrows
  anything: Workspace Access is forced on ("Public Workspace knowledge is required for Agents", 21 Aug
  2026), and Add from ClickUp adds emphasis. It doesn't build a wall. The only guard against answering about the
  wrong client is a sentence in Instructions naming the space it owns. Triggers only set when it runs.
- Give the smallest tool set that answers its question: read-only on systems of record, write only on
  the object it was handed. Money, agreements and rate cards get read tools and nothing else. With 20
  tools, the agent picks wrong and you can't debug it.
- Put anything long, per-client or changing in a Doc (Knowledge > Add from ClickUp > Docs) and tell the
  instructions to treat it as the source of truth.
- Give ambiguity somewhere to go: a verdict field needs a third value (Unclear) and an instruction not
  to fold it into the decisive one. An explicit excludes list on a signed-deliverables source fixed
  wrong verdicts that harder prompting never did.
- Add "never state a number you didn't read from a tool". An example in the instructions becomes a
  fact in the answer: one agent told to cite "Source: Client Memory X, dated 14 Aug 2026" stamped that
  date on sections with none. Write the shape with placeholders and add "a date only when that exact
  line carries one".
- The agent's one-line description shows in the agents list, so a demo agent's description follows the
  invented-data rule too. 2 agents sharing a first word make the `@` picker a coin toss: make the
  first word unique.

## Connect an MCP server

There's no API. It's all App Center (release 4.07, 18 Aug 2026). From `/<teamId>/settings/apps`, click
App Center: the row highlights on the first click, the modal opens on the second and fades in over
about 4 seconds, so a click during the fade dismisses it. Wait, then screenshot, then click. Under AI pick
MCP Servers:

- The catalogue (18 apps on 21 Aug 2026, each an OAuth into that account, so connecting one puts whatever
  it holds in front of the agent: never on a live account for a recording).
- Add Custom MCP Server, any URL: permissions (For all members or Just for me), then Name,
  URL, Description, Authentication (OAuth, Authorization header as `Bearer <token>`, or none).
  ClickUp then calls `tools/list` itself, and the discovered tools prove the connection.
- Pick the scope first. A personal connection has no credentials when its owner isn't logged in. A
  scheduled agent then doesn't error. It silently reports only what it reached. Anything scheduled needs the
  workspace connection.
- The rail shows a Custom MCP category only where a custom server is connected, the fastest proof of
  which workspace you're in. ClickUp's own card reading "Couldn't load tools" is an empty state and no fault.

Giving a connected server's tools to an agent is another screen. In the agent's Skills panel, Add
tools: the top-right Custom MCP Server button restarts the connect flow and would stand up a second
copy, so cancel it. The connected servers sit lower under Custom MCP Servers: click one, then Add
all or pick tools. The modal paints 4 to 6 seconds late, so click once, wait, screenshot. Search
returns tool groups ("status" gives Tasks and subtasks, 8 tools), not tools; a custom server's own
tools match by name. A Finish Setup banner saying Unavailable appears on some loads while the tools
work: ask the agent a question instead.

## Build one from scratch

Verified 21 Aug 2026. Don't brief the builder chat to create one: a 2.3k brief answered "Whoops!
Looks like we stumbled upon a hiccup in the matrix" and a 700-character one cleared the box and created
nothing, with no error. Use Start from scratch (top right). It opens the form on the right and the
builder chat on the left. Fields need different click modes:

- Name: one click, `cmd+a`, type, `Tab`. The browser tab title changing is the proof.
- Description: 2 clicks (the first only highlights the row and `cmd+a` then selects the page).
  It's a real `input[placeholder="Add description..."]`, so `fill()` works.
- Instructions: the second `contenteditable` on the page (the first is the builder chat). Type,
  don't paste: 2 pastes put in the wrong content or none. Type paragraph by paragraph with `Return`
  between them, avoid line-leading `1.` or `-` (becomes a list) and `/` or `@` (opens pickers). To
  append to existing instructions, open the field full screen (the arrows icon), `cmd+Down`, press
  `Return` twice to leave a bullet, type the new sections, close with X, then click Save changes.
- Saving is a banner: a Save changes bar appears and pushes everything below it down about 65px,
  so re-screenshot before every click. The toast is Agent saved successfully.
- A new agent can't create a task. It has one skill, ClickUp Default tools (14), and that has no Create
  task. An agent told to open a task will still say it did. Add tools > Popular tools > Create
  task, close, Save.
- Knowledge: narrow with Add from ClickUp when a workspace holds more than one client. Triggers
  default to Mention, Direct Message and Assign task, all on; Scheduled is added separately.

`Activate Super Agent` turns `Run` on; Send DM is the fastest full test and its reply is in a
thread. An agent creating something takes 50 to 60 seconds, and the card paints before the sentence.
A mention on a task is the cheapest run: type `@` plus the first word of the name, `Enter`, the
sentence, `cmd+Enter`. The answer arrives about 90 seconds later as a threaded reply
(`GET /api/v2/comment/<commentId>/reply`) and never as a comment. Then verify from the CLI:
`CU_TEAM_ID=<teamId> cu tasks --list <listId>` is the proof, and `cu task delete` cleans up before a demo.

An agent id seen in a network call is probably a different agent from the one you made: an open agent page polls
`.../agents/<id>/summary` for the agent it shows. Read `name` back, and use AI > All Agents, the
only full list (`/ai/agents` is the create screen).

## The Builder edits the prompt, never Knowledge

The Edit tab's "Super Agent Builder" rewrites instructions in natural language, and told on 24 Aug 2026
to add a second space it said: "I can't directly edit the Knowledge section (workspaceKnowledge)". That
is set by hand in the profile. Try the prompt first: an agent scoped to one space refused a question
about a company in another, and after the builder put the Companies list into its read order the same
question came back answered and sourced. Knowledge is access control; the reading order is usually the
real fault. The builder asks a clarifying question before writing, so a screenshot 20 seconds after
the instruction shows only the question.
