---
name: board
description: "Runs the ClickUp board lifecycle: move changes a task's status, spec turns an Open idea into a PRD, start implements a Ready task into a reviewed PR, ship merges a PR and closes the task. Use on /board-move, /board-spec, /board-start, /board-ship, or picking up, speccing or shipping a task."
argument-hint: move <task-id> <status> | spec <task-id> | start <task-id> [base-branch] | ship <task-id|PR-number> [integration-branch]
allowed-tools: Bash, Read, Edit, Write, Grep, Glob, Agent, Skill
tags: [drives, clickup, code]
lane: general
---

# /board

The board skill's verbs walk a task through its lifecycle: **move** (status only) → **spec** (Open → Ready PRD) → **start** (Ready → In Review PR) → **ship** (In Review → Done, merged). ClickUp is the source of truth for status throughout; GitHub holds the code, linked back to the task by the `CU-` id in the branch, PR and commits.

## Shared setup

- The `cu` CLI (alias `clickup`) must be authenticated: run `cu status` first. Token comes from `~/.config/clickup/config.json` or `CU_API_TOKEN`/`CU_TEAM_ID`. Missing or unauthenticated → stop, tell the user to run `cu init`, never guess a token.
- Read `.tasks/config.json` at the repo root for `list_id`, `status_map`, and the `github` owner/repo. It's the same file every skill and agent in this kit reads; its full shape is in the `clickup` skill's [`references/tasks-config.md`](../clickup/references/tasks-config.md). Map a friendly alias (`todo`, `ready`, `in-progress`, `in-review`, `done`) through `status_map`, which is list-specific and authoritative:

  ```jsonc
  // .tasks/config.json (example: Client Glance)
  "status_map": { "todo": "Open", "ready": "ready", "in-progress": "in progress", "in-review": "review", "done": "Closed" }
  ```

  If the file or `status_map` is absent, fall back to `todo`→`to do`, `in-progress`→`in progress`, `in-review`→`in review`, `done`→`complete`. Anything not in the map passes through as a literal ClickUp status name.
- Normalize a task id by stripping a leading `CU-` or `#`.
- `cu task get <id> --json` omits the description field. Read the PRD/AC body with `cu task get <id> --markdown`; parse `--json` output by grabbing the first `{`-prefixed line (`| grep -m1 '^{'`), since `cu` appends a `help[]` footer.
- ClickUp auto-links activity when a valid id appears anywhere in a PR title, body, branch name or commit message: `CU-{id}`, `#{id}`, or a custom id. A status tag with no space, `CU-{id}[status]`, moves the task when ClickUp ingests the commit: a valid fallback, but every verb below uses `cu task update` so the move is confirmed synchronously. Linking needs the repo's Space connected to GitHub (App Center → GitHub → Settings → repo → Add Space); if activity isn't showing on the task, check that mapping first.

## move

`/board move <task-id> <status>`: status change only, no code. `status` is a `status_map` alias or a literal ClickUp status name. Missing either argument → ask.

1. Parse and normalize the task id and status (lowercase, trim).
2. Resolve the status through `status_map` (above).
3. `cu task get <id> --json`, confirm the task exists, note the current status.
4. `cu task update <id> --status "<STATUS_NAME>" --json`. If the status doesn't exist in the task's list, surface the error and point at `status_map` or the list's real statuses.
5. Report: "Moved `<task-id>` (<name>) to <new status>."

## spec

`/board spec <task-id>`: writes the PRD into the task description, does not write code. No id given → list Open tasks (`cu tasks --list <id> --status Open`) and ask which.

1. Read the raw idea: `cu task get <id> --markdown` (body/PRD) plus `cu comments list <id>`. Capture the original text verbatim; it survives at the bottom of the PRD.
2. Investigate the codebase before writing, and cite real paths, file counts and duplication. Rule out duplication first: search the board across every status (`cu tasks --list <id> --json` and `--closed --json`) for a near-identical title, and check whether the code already implements it. Either case: don't write a build PRD, report the match and recommend `move` to In Review/Done instead.
3. Draft the PRD in this shape, nothing speculative:

   ```markdown
   ## Problem
   What hurts today, with code evidence.
   ## Goals
   What done looks like, 2-4 bullets.
   ## Non-goals
   What this task deliberately skips.
   ## Acceptance Criteria
   - [ ] Verifiable, with the check command where one exists.
   ## Technical sketch
   Named files/modules, a sketch not a design doc.
   ## Risks & open questions
   Ambiguity and deferred decisions.
   ---
   *Original idea:* <verbatim>
   ```

4. A blocking ambiguity (two incompatible reads, a missing business call) → ask before writing. Otherwise record it under Risks and proceed.
5. `cu task update <id> --description "$(cat <prd-file>)" --markdown`, then run `move` to Ready.
6. Report: task id/name Open → Ready, a one-paragraph PRD summary, open questions.

Notes: spec the task, not the epic: 3+ independent ideas means offering a split (`cu task create`) instead of one bloated PRD. Reuse existing board tags only (`cu task get <id> --json | jq -r .tags` across tasks): a new tag fragments the taxonomy. `task update` and `task create` take different flags: `--description-file` and `--tag` exist on `create` only, `update` rejects both outright: pass a long PRD through the shell (`--description "$(cat <path>)" --markdown`), and tag an existing task at `create` time or via `POST /api/v2/task/<id>/tag/<tag_name>` (needs the owner token, not the team one). Never invent AC the idea doesn't imply.

## start

`/board start <task-id> [base-branch]`: implements the fix in an isolated worktree and opens the PR. No id given → take the highest-priority Ready task (`cu tasks --list <id> --json`) you can actually satisfy; skip one whose AC is a decision only the user can make (a brand or pricing call), say which you took and why, and only ask if nothing in the list is implementable.

1. Read the task: `--json` for status/tags, `--markdown` for the description/AC (json omits it). Thin description → check comments for a linked GitHub issue (`gh issue view <n> --repo <owner>/<repo>`) or `.tasks/**` for a spec file. Restate your understanding in 1-2 sentences before coding; ambiguous or missing AC → stop and ask. Read the AC for its kind: *build it* (write, test, ship: the default); *observe it* (behavior already exists, the deliverable is a gate that drives the real path and fails when it breaks); *measure it* (the AC names a number and a substrate: take the reading on that substrate, report a simulator number as a floor, not the answer).
2. `move` to In Progress.
3. Resolve the integration branch: the `base-branch` arg, else `staging` if it exists, else the repo default (`gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'`). An explicit `staging` that doesn't exist → stop and ask, never silently retarget.
4. Create a worktree from `origin/<integration-branch>` (never the local branch: another session's unpushed commit would ride along):

   ```bash
   git fetch origin
   git worktree add -b CU-<task-id>_<short-slug> ../<repo>-CU-<task-id> origin/<integration-branch>
   ```

   Branch name must contain `CU-<task-id>` for ClickUp's link. Put the worktree beside the repo (`../<repo>-CU-<id>`), not under `/tmp`: a sibling-relative build path resolves to nothing there. The worktree isn't private: another session's tooling can write into it (a lockfile bump from nothing you did). Never `git add -A` or `commit -a`; name the files you touched and leave the rest.
5. Implement the smallest change that satisfies the AC. Read the project's `CLAUDE.md` and path-scoped `.claude/references/` for files you touch. Add or adjust a test that fails before, passes after, where testable.
6. Verify inside the worktree (build, tests, lint) after your last edit: a run kicked off mid-edit proves nothing. A gate that failed twice for ordering (not for the change) gets repaired in the target, not retried a third time.
7. Commit only the files you touched:

   ```
   <concise summary>

   CU-<task-id>
   ```

   No `Co-Authored-By`, whatever the session defaults say: these are signed by the board owner. Add `Closes #<n>` if it resolves a linked issue.
8. Push and open the PR, title must contain `CU-<task-id>`:

   ```bash
   git push -u origin CU-<task-id>_<short-slug>
   gh pr create --base <integration-branch> --head CU-<task-id>_<short-slug> \
     --title "CU-<task-id> <task name>" --body "<body>"
   ```

   Body restates the AC, summarizes the change, lists verification, references any linked issue, ends with `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
9. Self-review: light pass against the AC; run `/code-review` on the branch diff if available and fix high-confidence findings. This is a sanity check, not `ship`'s deep gate.
10. `move` to In Review.
11. Report: PR URL and branch, worktree path (for `ship`), what was implemented mapped to each AC, verification results, integration branch targeted.

Notes: never move to In Review on failing checks or unmet AC: report the gap. Tick what you met, name what you didn't with the reason in one clause (a locked phone, a person who has to speak, a permission no script can flip is finished work honestly reported; writing it as met is what makes the board useless). Where the code already does better than the AC's words say, say so instead of changing the code to match them.

## ship

`/board ship <task-id | PR-number> [integration-branch]`: the quality gate between In Review and Done. **Done means merged and verified**; a status flip alone is never shipping. No argument → list In Review tasks (`cu tasks --list <id> --status "<status_map.in-review>" --json`) and ask which.

1. Resolve the task ↔ PR pair. Task id given → find the PR via `gh pr list --search "CU-<id> in:title" --state all` (also try `in:body` and the branch name), or check `cu task get <id>` comments for a linked PR. PR number given → `gh pr view <n>` and pull the `CU-` id from title/body/branch. No PR found → stop, suggest `start` first. Capture PR number, head/base branch, task id, repo.
2. Read every AC from the task description (`--markdown`, json omits it) and any linked issue; `gh pr diff <n>` for the change itself.
3. **AC gate**: prove each criterion from the diff and current code: grep to confirm a field's gone, check the server logic for pagination, find and run a claimed test, flag "renders correctly" as needing manual verification. Run the real tests and build against the PR branch. Record anything unmet or untestable.
4. **Quality gate**: clean, DRY, integrates elegantly. Run `/code-review` at `high` (`ultra` for large/risky diffs) if available, plus: correctness (logic, edge cases, races); DRY (checks `@heyramzi/*` and local helpers before calling something new); clean (no dead code, debug output, commented-out blocks); fits the surrounding style and the integration branch's current state (pull latest, check conflicts); project rules (`.claude/references/`, lint, type-safety); regression risk.
5. Verdict: **PASS** (merge) / **CLEANUP-THEN-PASS** (apply safe fixes on the branch, push, re-verify, then merge) / **FAIL** (don't merge; report what's missing, comment on the PR, optionally `move` back to In Progress). When in doubt, FAIL.
6. Make the merge clean: `git fetch origin`, rebase or merge the PR branch onto the latest integration branch, resolve conflicts, re-verify, push.
7. Squash-merge, carrying the `CU-` id and done tag so ClickUp can auto-close:

   ```bash
   gh pr merge <pr-number> --squash --delete-branch \
     --subject "<concise summary>" --body "CU-<task-id>[<status_map.done>]"
   ```

   Confirm with `gh pr view <n> --json state,mergedAt`. A linked `Closes #<n>` issue closes too.
8. `move` to Done as the authoritative change (don't rely on the commit tag alone). Comment on the task (`cu comments add <id> --text "..."`) with what shipped, which AC were verified, what the review covered, and any manual-verification items.
9. Clean up: `git worktree remove <path>` (left by `start`); confirm the remote branch was deleted by the squash-merge.
10. Report: verdict, what the review covered, the merge commit, task moved to Done, manual follow-ups.

Notes: this is a code-quality gate, not just an AC checker, which would verify AC without merging. Watch for duplicate tickets surfacing from the PR (a restated sub-scope, an accidental clone): close the duplicate with a comment pointing at this one. Never merge with failing checks, unmet AC, or unresolved findings.
