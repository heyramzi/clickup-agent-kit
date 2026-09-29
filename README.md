# ClickUp Agent Kit

Hand your ClickUp workspace to an AI agent. It reads every space, list and task,
tells you what's broken, and fixes it when you say so.

`cu` is the command your agent runs. It has 215 of them in a single file with no
runtime dependencies. The skills in this repo teach your agent how to drive it.

The kit is free. It sits next to the [ClickUp Foundations](https://go.upsys-consulting.com/skool)
course, which shows you how to use it. The source stays private, so this repository
only carries the built bundle.

## Install

```
npm i -g heyramzi/clickup-agent-kit
cu init
cu skills install
```

`cu init` asks for a ClickUp API token, which you make under your avatar, then
**Settings**, then **Apps**. Nothing leaves your machine except calls to ClickUp.

## The first three commands

```
cu status                       the workspace your token actually reached
cu hierarchy                    every space, folder and list in one screen
cu fields list --list <listId>  the custom fields on a list, and their types
```

Then find work without opening the app. A comma list matches any of its values:

```
cu tasks query --list <listId> --status "in progress,review" --due-before 2026-10-31
cu tasks query --space <spaceId> --tag urgent --assignee <userId>
cu docs search --parent <spaceId> --parent-type SPACE
```

`cu --help` prints all 215.

## Skills for your AI agent

The repo carries the `clickup` and `board` skills next to the binary. They work on any ClickUp board, with no special
fields to set up.

- `clickup` teaches your agent the `cu` CLI: tasks, docs, hierarchy, bulk work, and what to do in
  the browser when ClickUp has no API for it.
- `board` runs a task from idea to merged PR. It writes the brief, builds it, then ships it.

```
cu skills install            into ./.claude for this project
cu skills install --global   into ~/.claude for every project
```

It copies the files, so run it again after `npm i -g heyramzi/clickup-agent-kit` to update them,
and keep your own edits in separate skills.

The `board` skill reads a small cheat sheet, `.tasks/config.json`, at the root of your project. It
holds your list id, your real status names and your GitHub repo. Only `list_id` and `status_map`
are needed to start, and your workspace ids stay in your own copy. Nothing here knows them.

A good first win: pick a real task on your board and say "write the brief for this task". The
agent reads the task, writes the brief into it, and you have a spec it can build from.

## Planning the week, and turning calls into tasks

These come with [Agency Master](https://www.upsys-consulting.com/en/agency-master), the ClickUp
operating system. `batch-workload` plans your team into weekly batches and keeps everyone under a
points cap. `transcript-to-tasks` turns a call into owned, dated tasks. Two agents run them:
`clickup-pm` and `capacity-planner`. They plan in batches with points, and those fields come with
the Agency Master template.

## One safety rule

**Read commands are safe. A write lands with no confirmation.** `cu list delete`
deletes the list and every task in it, straight away. Stay on the read commands
for a week before you run a write, and read `cu <command> --help` first.

## Using it with Claude

Point your agent at the CLI and let it choose the command:

```
I have the `cu` CLI installed and pointed at my ClickUp workspace.
Run `cu --help`, then `cu hierarchy`, and tell me the one structural
problem you can see. Only read commands. Ask me before any write.
```

## Requirements

Node 20 or newer.
