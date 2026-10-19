# The board

survey and reconcile read and repair the whole board. The rest walk one task through its
life: move (status only), spec (Open to Ready, a PRD), start (Ready to In Review, a PR),
ship (In Review to Done, merged). ClickUp owns status; GitHub holds the code, linked by the `CU-`
id in the branch, PR and commits.

## Shared setup

- Run `cu status` first. Unauthenticated: stop and tell the user to run `cu init`.
- Read `.tasks/config.json` for `list_id`, `status_map` and `github` ([`tasks-config.md`](tasks-config.md)).
  Map an alias (`todo`, `ready`, `in-progress`, `in-review`, `done`) through `status_map`, which is
  list-specific and authoritative. Without it: `to do`, `in progress`, `in review`, `complete`.
  Anything not in the map passes through as a literal status.
- Strip a leading `CU-` or `#` from a task id.
- Read the body with `cu task get <id> --markdown`: `--json` omits the description.
- ClickUp links activity when `CU-{id}`, `#{id}` or a custom id is in a PR title, body, branch or commit,
  and `CU-{id}[status]` moves the task. These verbs all use `cu task update` instead, so the move is
  confirmed. No activity on the task means the repo's Space isn't connected (App Center > GitHub).

## survey

Writes nothing. Run it when asked about the board, what's late or what's next.

1. `cu tasks --list <list_id> --json`, grouped by status through `status_map`.
2. Add `cu tasks query --list <list_id> --due-before <today> --json` and every task with no owner. Past
   its date with the status unmoved is late, whatever the comments say. Count first, then the list.
3. One next action per column: spec what's Open, start what's Ready and ship what's in review. A late task gets an
   owner and a reason, or a new date.
4. Recommend one task to do first, with a sentence of why.

Done when the report has the late count, the per-column actions and the one pick.

## reconcile

Fixes the board after work moved and the board didn't. Writes, so it never runs by accident.

1. Inventory every open task with its full description (`--markdown`).
2. Check each against evidence (the file, doc, git log, last transcript), not memory.
3. Sort: KEEP, MOVE (target via `status_map`), SPLIT, or STALE (the description describes a gone world).
4. Find the gap: work happening with no task behind it, the half a status sweep misses.
5. Show the plan as a table, apply it, and comment on each moved task with the line of evidence.

A STALE task gets its description rewritten before it moves, or the wrong spec hides in a later column.
Close only what you can quote proof for. Done when every write reads back with `cu task get <id>`.

## move `<task-id> <status>`

Status only. Resolve the status through `status_map`, `cu task get <id> --json` to confirm the task and
its current status, then `cu task update <id> --status "<NAME>" --json`. A status missing from the
task's list is surfaced with the list's real statuses. Done when you've said "Moved `<id>` to <status>."

## spec `<task-id>`

Writes the PRD into the description and no code. No id: list Open tasks and ask.

1. Read `cu task get <id> --markdown` and `cu comments list <id>`. The original text survives verbatim at
   the bottom of the PRD.
2. Investigate the code first and cite real paths and counts. Rule out duplicates: search every status
   (`cu tasks --list <id> --json` and `--closed`) for a near-identical title, and check whether the code
   already does it. Either way, don't write a build PRD: report the match and recommend `move` to Review or Done.
3. Draft, nothing speculative: `## Problem` (with code evidence), `## Goals` (2-4), `## Non-goals`,
   `## Acceptance Criteria` (`- [ ]`, with the check command where one exists), `## Technical sketch`
   (named files), `## Risks & open questions`, then `---` and `*Original idea:*`.
4. A blocking ambiguity (2 incompatible reads, or a missing business call): ask before writing. Otherwise
   record it under Risks and go on.
5. `cu task update <id> --description "$(cat prd.md)" --markdown`, then `move` to Ready.

Spec one task and never the epic. 3 or more independent ideas get a split offer (`cu task create`). Reuse
existing tags only. `update` takes neither `--description-file` nor `--tag`; tag at `create`, or
`POST /api/v2/task/<id>/tag/<name>` (owner token). Don't invent AC the idea doesn't imply. Done when the
report gives Open to Ready, a PRD paragraph and the open questions.

## start `<task-id> [base-branch]`

Builds in a worktree and opens the PR. No id: take the highest-priority Ready task you can satisfy; skip
one whose AC is a decision only the user can make (a brand or pricing call), and say which you took.

1. Read it (`--json` for status and tags, `--markdown` for AC). A thin body: check comments for a linked
   issue (`gh issue view <n> --repo <owner>/<repo>`) or `.tasks/**`. Restate it in 1-2 sentences; missing
   AC means stop and ask. Read the AC's kind: *build it* (default), *observe it* (the deliverable is a
   gate that fails when the real path breaks), *measure it* (take the reading on the named substrate; a
   simulator number is a floor).
2. `move` to In Progress.
3. Integration branch: the argument, else `staging` if it exists, else the repo default
   (`gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`). An explicit `staging` that's
   missing: stop and ask, never retarget.
4. Worktree from `origin/<branch>`, never the local branch (another session's unpushed commit would ride
   along), beside the repo, not under `/tmp` (sibling-relative build paths resolve to nothing there):

   ```bash
   git fetch origin
   git worktree add -b CU-<id>_<slug> ../<repo>-CU-<id> origin/<integration-branch>
   ```

   Another session's tooling can write into it (a lockfile bump you didn't make). Name the files you
   touched; never `git add -A` or `commit -a`.
5. Implement the smallest change that meets the AC, following the project's `CLAUDE.md` and
   `.claude/references/` for those files, with a test that fails before and passes after where testable.
6. Verify in the worktree after your last edit (a run mid-edit proves nothing). A gate that failed twice
   for ordering gets repaired in the target, not retried a third time.
7. Commit only your files: a summary, a blank line, `CU-<id>` (plus `Closes #<n>` if it resolves an
   issue). No `Co-Authored-By`: these are signed by the board owner.
8. `git push -u origin CU-<id>_<slug>`, then `gh pr create --base <branch> --head CU-<id>_<slug> --title
   "CU-<id> <task name>" --body ...`. The body restates the AC, summarizes the change, lists verification
   and ends with `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
9. Self-review against the AC, `/code-review` on the diff if available, fix high-confidence findings.
10. `move` to In Review.

Done when the report has the PR URL, branch, worktree path, each AC with its result, and the branch
targeted. Never move to In Review on failing checks or unmet AC. Tick what you met, name what you didn't
with the reason in a clause (a locked phone or a permission no script flips is honest, and writing it as met
breaks the board). Where the code already beats the AC's words, say so and leave the code alone.

## ship `<task-id | PR-number> [integration-branch]`

The gate between In Review and Done. Done means merged and verified. No argument: list In Review
tasks (`--status "<status_map.in-review>"`) and ask.

1. Resolve the task and PR pair: `gh pr list --search "CU-<id> in:title" --state all` (also `in:body`
   and the branch), or the task's comments; from a PR, `gh pr view <n>` and the `CU-` id in title, body
   or branch. No PR: stop and suggest `start`.
2. Read every AC (`--markdown`) and any linked issue, and `gh pr diff <n>`.
3. AC gate: prove each criterion from the diff and current code (grep that a field is gone, read the
   pagination logic, run the claimed test; "renders correctly" is a manual item). Run the real tests and
   build on the PR branch.
4. Quality gate: `/code-review` at `high` (`ultra` for big or risky diffs) if available, plus
   correctness (edge cases, races), DRY (check `@heyramzi/*` and local helpers first), clean (no dead
   code, debug output or commented blocks), fit with the integration branch as it is now, project rules,
   regression risk.
5. Verdict: PASS (merge), CLEANUP-THEN-PASS (safe fixes on the branch, push, re-verify, merge) or
   FAIL (comment on the PR, maybe `move` back to In Progress). In doubt, FAIL. Never merge with
   failing checks, unmet AC or open findings.
6. Rebase or merge onto the latest integration branch, fix conflicts, re-verify, push.
7. Squash-merge carrying the id and done tag so ClickUp can close it:

   ```bash
   gh pr merge <pr> --squash --delete-branch --subject "<summary>" --body "CU-<id>[<status_map.done>]"
   ```

   Confirm with `gh pr view <n> --json state,mergedAt`.
8. `move` to Done (the authoritative change), and comment on the task: what shipped, which AC were
   verified, what the review covered, manual items.
9. `git worktree remove <path>`; confirm the remote branch is gone.

Done when the report has the verdict, merge commit, Done status and manual follow-ups. A duplicate
ticket surfacing from the PR gets closed with a comment pointing here.
