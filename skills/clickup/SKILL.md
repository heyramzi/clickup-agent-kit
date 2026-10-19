---
name: clickup
description: "Runs ClickUp work: the cu CLI, bulk scripts, browser-only surfaces and the board. Use on any ClickUp task, doc or bulk update, 'what's late', or to spec, start or ship a task."
tags: [drives, clickup, browser]
lane: general
---

# ClickUp

3 ways in, in this order: the cu CLI first (tasks, docs, hierarchy, time, and the private
frontdoor surfaces it wraps), scripts against the REST API for bulk data work, and the browser
only for what neither reaches.

This skill is the base of a small free kit. All pieces read `.tasks/config.json` first: the list id,
the real status names and the GitHub repo. Shape and rules:
[`references/tasks-config.md`](references/tasks-config.md).

| Piece | Kind | Job |
| --- | --- | --- |
| `clickup` | skill | Drive ClickUp, and walk a task from idea to merged PR |

Planning the week in batches with points, and turning a call into tasks, come with Agency Master.

## Pick the branch

| The job | Read |
| --- | --- |
| What's late, spec, start or ship a task | [`board.md`](references/board.md) |
| Any `cu` command, auth, tokens, task, comment or chat write | [`cu-commands.md`](references/cu-commands.md) |
| Custom fields, merging them, views, tags | [`fields.md`](references/fields.md) |
| Statuses, templates, automations, `cu net capture` | [`frontdoor.md`](references/frontdoor.md) |
| Docs, doc renames, call transcripts, bulk REST scripts | [`bulk-and-docs.md`](references/bulk-and-docs.md) |
| Super Agents, MCP servers | [`super-agents.md`](references/super-agents.md) |
| A click path, a screenshot, the Workload view | [`browser.md`](references/browser.md) |

Read the file before a write you haven't made once, because each entry in them cost a run. Config values
come from `.tasks/config.json`, never from memory.

## The board

`/clickup board <verb>`. `survey` reports what's late and next and writes nothing. `reconcile`
makes the board match the work, with evidence for each move. `move` changes a status, `spec`
turns an Open task into a PRD, `start` builds it in a worktree to a reviewed PR, and `ship` gates,
merges and closes it. Done means merged and verified. A status flip doesn't count.

## cu CLI

One global binary, `cu <command>`, with no `clickup` alias. `cu --help` is the command list and the
binary wins over any table. `cu init` sets up, `cu status` shows auth.
Install with `npm i -g heyramzi/clickup-agent-kit`, then `cu init`. It's a public repo, so there's no
token to ask anybody for. Course: ClickUp Foundations at
[skool.com/ai-agency-systems-3191](https://www.skool.com/ai-agency-systems-3191).

The traps that cost the most:

- `task get --json` drops the description and custom fields. `--markdown` is the full body.
- A body with a heading, list or table goes in `markdown_content` (`--markdown` plus
  `--description "$(cat f.md)"`). `description` is plain text and returns 200 either way.
- Dates are epoch milliseconds. A date string lands on the wrong day without an error.
- Whole scopes are missing on a member token (tags, custom-field writes): use the owner token.
  A `401 ACCESS_081` means the other token owns the object. It doesn't mean you're forbidden.
- Frontdoor commands (templates, automations, statuses, dashboards, agents, field merge) read a
  captured browser session, not `pk_`. Run `cu net capture` recently, against the target workspace.
- Verify every write by reading it back. A 200 isn't proof: guests, ClickApps that are off and
  automation-owned lists all return success and change nothing.

Done when the change reads back through `cu` the way you meant it.

## When REST has no route

- Workspace-wide search and reminders have no REST route. ClickUp's official MCP server has a
  search tool. Without the AI add-on, Unlimited gets 300 MCP calls a day, so use it for one lookup,
  never a loop. Where the plan has no daily cap, the MCP shares the REST per-minute budget.
- Creating and editing custom fields go through routes ClickUp doesn't document
  (`POST /list/{id}/field`, `PATCH /field/{id}`). They can change without notice, so read the field
  back after every write.

## Bulk data work

Scripts against REST with `curl` or Python, tokens from `.env.local`, one per workspace (the wrong one
401s on objects that exist). Patterns, rate limits and the demo-workspace data bar:
[`bulk-and-docs.md`](references/bulk-and-docs.md).

## Browser only

A last resort, because a click path breaks on a redesign and an API call doesn't. Announce "I'm using
the clickup skill, browser mode."

1. Record the call once, script it forever. `cu net capture <url>` learns read endpoints,
   `cu net record` learns a write flow, `cu net replay` re-issues either (writes need `--allow-write`).
2. `ego-browser` first, Claude in Chrome as the fallback. `wait()` takes seconds, not milliseconds.
3. Demos are invented, always, in their own list. Never a real client's board with the logo swapped,
   even briefly: the intermediate state ends up in a take.4. Verify with the API. A screenshot proves a dialog closed and nothing about whether an
   automation fired.

Done when the object reads back through `cu`. Details: [`browser.md`](references/browser.md),
[`frontdoor.md`](references/frontdoor.md), [`super-agents.md`](references/super-agents.md).
