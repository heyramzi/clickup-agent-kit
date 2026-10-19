# `.tasks/config.json`: the map agents read

It sits at the root of the folder you open Claude Code in. ClickUp holds the work, and this file is the
map to it. It says which list is the board, what its statuses are named and which repo the work ships to.
Without it an agent guesses a status name and rediscovers the list id every run.

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
  "status_map": { "todo": "to do", "ready": "ready", "in-progress": "in progress", "in-review": "in review", "done": "complete" },
  // Only the board's start and ship verbs read this.
  "github": { "owner": "your-org", "repo": "your-repo" }
}
```

Only `list_id` and `status_map` are needed to start. Agency Master adds `fields` and `team` for its
planning skills.

Set it up with 3 commands. `cu status` shows auth and the workspace. `cu hierarchy` gives the list id you work
in. `cu list get <listId>` gives the real status names.

- Read it first, every run. A skill that asks ClickUp for an id the config holds starts a second map.
- When a key is missing, find it with `cu`, use it, and write it back in the same run.
- Resolve a status through `status_map`; anything not in the map goes through as the literal name.
  Fix a renamed status here. Don't work around it.
- It maps and doesn't store work. Task lists, weekly plans and notes live on the board.

Done when `cu status` passes and `list_id` resolves with `cu list get`.
