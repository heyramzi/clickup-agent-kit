# `.tasks/config.json`: the one map every agent reads

Every skill in this kit reads the same file before it touches ClickUp. It sits at the root of the
folder you open Claude Code in, a repo or a plain workspace folder. ClickUp holds the work. This
file holds the map to it: which list is the board, what its statuses are really called, and which
GitHub repo the work ships to.

That's the whole reason it exists. Without it, an agent guesses a status name and rediscovers the
list id on every run. With it, everything reads the same ids, and it works on any ClickUp board:
no special fields needed.

## The shape

```jsonc
{
  "tracker": "clickup",
  "cli": "cu",
  "team_id": "9012345678",            // the workspace
  "list_id": "901200000001",          // the board this folder works on
  "list_name": "Delivery",
  "task_id_prefix": "CU-",
  "task_url": "https://app.clickup.com/t/{id}",

  // Friendly alias -> the real status on this list. Statuses differ per space.
  "status_map": {
    "todo": "to do",
    "ready": "ready",
    "in-progress": "in progress",
    "in-review": "in review",
    "done": "complete"
  },

  // Only the board skill's start and ship verbs read this.
  "github": { "owner": "your-org", "repo": "your-repo" }
}
```

Only `list_id` and `status_map` are needed to start. `github` is only there if you ship code from
the board.

Agency Master adds `fields` (the BATCH and Points ids) and `team` (who's on the team and how much
each person carries a week), for the planning skills that come with it.

## Readers

| Key | Read by | Written by |
| --- | --- | --- |
| `list_id`, `status_map` | every skill | you, once, at setup |
| `github` | `board` start and ship | you, once |

## Setting it up

3 commands, about 2 minutes:

```bash
cu status                                  # auth works, and which workspace you're on
cu hierarchy                               # copy the list id you actually work in
cu list get <listId>                       # the real status names, for status_map
```

## One source of truth

- **Read it first, every run.** A skill that asks ClickUp for a list id the config already has is
  the start of two maps.
- **Discover once, then write it back.** When a key is missing, find the value with `cu`, use it,
  and add it to the file in the same run. The next agent shouldn't pay for the same lookup.
- **Never hard-code a status in a prompt or a skill.** Resolve the alias through `status_map`.
  Anything not in the map goes through as the literal ClickUp name.
- **The config maps, it doesn't store work.** No task lists, no weekly plans, no notes in here.
  Those live on the board, where the team can see them.
- **A wrong value is fixed here, not worked around.** If a status got renamed in ClickUp, change
  `status_map`, and every agent picks it up on its next run.
