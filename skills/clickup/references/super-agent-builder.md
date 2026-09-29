# Building a Super Agent

Filling the form from scratch and editing it afterward, both verified end to end. Setup: [`references/super-agents.md`](super-agents.md).

## Build a Super Agent from scratch

Verified 21 Aug 2026 in a demo workspace.

**Do not brief the builder chat.** The prompt box on `/<teamId>/ai/agents`
("Describe tasks or workflows that need automating") answered a 2.3k brief with
*Whoops! Looks like we stumbled upon a hiccup in the matrix*, and a 700-character
one by clearing the box and returning to the start page with no error at all.
Neither attempt created anything — and neither told the truth about that either.
An earlier note here said the chat was more reliable than the form; that is now
wrong, and the form path below is what works.

**An agent id you find in a network call is probably not the one you just made.**
An open agent page polls `.../agents/<id>/summary` for whatever agent it is
showing, and a workspace usually holds several. Read the `name` back before
acting on the id: in one workspace `8cbypq9-109195` looked like a fresh draft
and was a long-standing **Client Profitability** agent. Confirm against
**AI → All Agents**, the only place the full list lives; `/ai/agents` is the
create screen, not the list.

**Start from scratch**, top right, opens `/<teamId>/ai/agents/<agentViewId>` with
the full form on the right and the builder chat on the left. Fill the form. Every
field needs its own click to enter edit mode, and the clicks are not the same:

- **Name.** One click turns the title into an inline input. `cmd+a`, type, `Tab`
  to commit. The browser tab title changes to the new name, which is the cheapest
  proof it took.
- **Description.** Needs **two** clicks. The first only highlights the row and
  leaves focus outside, and a `cmd+a` there selects the whole page instead of the
  field, so the next paste lands nowhere.
- **Instructions.** One click inside the grey box gives a caret. **Type, do not
  paste.** Two pastes in this session put the wrong content in the field, once a
  stale clipboard, once nothing at all, and a paste that fails looks exactly like
  a click that missed. Typing the paragraphs one by one with `Return` between
  them is slower and it is the only version that landed. Avoid line-leading `1.`
  or `-`, which the editor turns into lists, and avoid `/` and `@`, which open the
  tool and mention pickers mid sentence.

**Editing an agent that already has instructions is a different job from filling an
empty one, and it works.** Open the field full screen with the ⤢ icon beside
**Instructions**, click anywhere in the body, `cmd+Down` to reach the very end. If
the last thing is a bullet, press `Return` twice: the first makes a new bullet,
the second exits the list so a plain paragraph follows. Type the new sections,
one `type` call per paragraph with a `Return` between them — verified 21 Aug 2026
appending six sections to Client Profitability, all landed intact. Close with the
modal's **X**; the **Save changes** banner is waiting behind it and still needs
its own click.

**Re-screenshot before every click in this form.** Saving is a banner, not a
button: the moment a field changes, a **Save changes** bar appears between the
header and Instructions and pushes everything below it down by about 65px, so a
coordinate read before the bar appeared now lands one row too high. `Save` gives
the toast **Agent saved successfully**.

**Skills is not what you think it is.** A new agent has one skill, ClickUp
**Default tools**, and hovering the `14 tools` chip lists them: Create schedule,
Edit self, Execute code, Generate image, Load assets and objects, Load Custom
Fields, Post reply, Retrieve Chat messages, Retrieve task list, Search activity,
Search users and teams, Search Workspace, Transcribe media, Write a to-do list.
**Creating a task is not among them.** An agent told to open a task will say it
did and will not have. Add it: **Add tools** → Popular tools → **Create task**,
which flips to *Added*, then close the modal and Save.

**Knowledge** is three things. **Workspace Access** is a single toggle for the
whole workspace and it is on by default. **Add from ClickUp** narrows it, with
Spaces & Lists / Tasks / Docs / Chats behind it. **External Search** carries Web
Search, ClickUp Help Center and GitHub, all off. Narrow to the space when the
workspace holds more than one client's data, or the agent will happily answer
about the wrong one.

**Triggers** default to three manual jobs, all on: Mention, Direct Message, Assign
task. **Scheduled** is added separately.

**An example in the instructions becomes a fact in the answer.** The Client Brain
agent was told to cite its source "in this shape: Source: Client Memory Vaqueros
Tex-Mex, standing preference dated 14 Aug 2026". It then stamped *14 Aug 2026* on
answers taken from sections that carry no date at all, which is exactly the
fabrication the agent existed to prevent. Write the shape with placeholders and
no real values, and add the negative rule beside it: add a date only when that
exact line carries one, never reuse a date from another line. Re-testing after
that edit produced clean citations naming two sections and no date.

`Activate Super Agent` turns `Run` on. `Run` offers **Send DM** and a preview of
the scheduled run. The DM is the fastest end-to-end check, and its reply lands
in a thread, so open the thread rather than expecting it in the main pane. A DM
that makes the agent create something takes **50 to 60 seconds**, and the task
card paints in the thread before the sentence does, so a thread showing a card
and "No replies" is still working, not stuck.

**Then verify from the CLI.** The card in the chat is the agent's own claim.
`CU_TEAM_ID=<teamId> cu tasks --list <listId>` is the proof, and the same
command deletes the test artefacts before a live demo.

**Two agents whose names share a first word make the mention picker a coin
toss.** Typing `@Upsell` in a task comment box returned both `Upsell
Intelligence` and `Upsell Agent` under the Agents tab. Name a demo agent so its
first word is unique in the workspace, or rename the other one before
recording. Measured 18 Sep 2026 building the generic Upsell Agent beside the
Upcut copy.

**A mention on a task is the cheapest run, and its answer is a threaded reply,
not a comment.** Type `@` plus the first word of the agent's name in the task's
comment box, `Enter` to take the suggestion, then the sentence, then
`cmd+Enter`. The answer lands about 90 seconds later **under that comment**, so
`GET /api/v2/task/<id>/comment` still returns the same count and reads as
failed: the reply is at `GET /api/v2/comment/<commentId>/reply`, and the thread
stays closed until somebody clicks `1 reply`. Measured 18 Sep 2026 building
Upsell Intelligence in the Upcut demo.

**The form's fields answer to different tools.** The description is a real
`input[placeholder="Add description..."]`, so `fill()` writes it in one call;
the name takes a click then `cmd+a` and typing; the instructions box is the
second `contenteditable` on the page (the first is the builder chat), so
`document.querySelector('[contenteditable="true"]')` reports the instructions
empty when they are fine.

## The Super Agent Builder edits the prompt, never the Knowledge

The Edit tab on a Super Agent is a chat with a "Super Agent Builder" that rewrites the
agent in natural language. It will happily restructure the instructions, and it says so:
asked on 24 Aug 2026 to give 🧠 Client Brain access to a second space, it answered that it
could handle the instructions but **"there's a platform limitation: I can't directly edit
the Knowledge section (workspaceKnowledge) from here."** That section is set by hand in the
agent profile.

In practice the instruction rewrite was enough. The agent had been scoped to DELIVERY 3.0
and refused a question about a company in SALES 3.0, naming the record it had found and
ignored. After the builder inserted the Companies list into its read order, the same
question came back answered and sourced. **So try the prompt first and re-ask the question
before going near the profile**, because the Knowledge section is access control rather
than the agent's reading order, and the reading order is usually what is actually wrong.

The builder asks a clarifying question before it writes, and waits. A run that fires the
instruction and screenshots twenty seconds later catches the question, not the change.
