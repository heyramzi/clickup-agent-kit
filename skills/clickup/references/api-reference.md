# ClickUp REST API patterns

## Contents

- Core Patterns
- Key Endpoints

## Core Patterns

### Bash one-liners

```bash
CU_TOKEN="pk_..."

# GET
curl -s -H "Authorization: $CU_TOKEN" "https://api.clickup.com/api/v2/space/SPACE_ID/view"

# UPDATE task
curl -s -X PUT -H "Authorization: $CU_TOKEN" -H "Content-Type: application/json" \
  "https://api.clickup.com/api/v2/task/TASK_ID" \
  -d '{"name": "New Name", "description": "...", "status": "in progress"}'

# DELETE view
curl -s -X DELETE -H "Authorization: $CU_TOKEN" "https://api.clickup.com/api/v2/view/VIEW_ID"
```

### Python bulk script pattern

```python
import subprocess, json

TOKEN = "pk_..."

def api(path, method="GET", body=None):
    args = ["curl", "-s", "-H", f"Authorization: {TOKEN}"]
    if method != "GET":
        args += ["-X", method]
    if body is not None:
        args += ["-H", "Content-Type: application/json", "-d", json.dumps(body)]
    args.append(f"https://api.clickup.com/api/v2{path}")
    r = subprocess.run(args, capture_output=True, text=True)
    return json.loads(r.stdout) if r.stdout.strip() else {}

def update(task_id, data):
    return api(f"/task/{task_id}", "PUT", data).get("name", "ERR")

def create(list_id, data):
    return api(f"/list/{list_id}/task", "POST", data).get("name", "ERR")
```

## Key Endpoints

### Hierarchy discovery

```
GET /team/{workspaceId}/space               → list all spaces
GET /space/{spaceId}/folder                 → folders in space (include nested lists)
GET /space/{spaceId}/list                   → folderless lists
GET /folder/{folderId}/list                 → lists in folder
GET /list/{listId}/task?include_closed=true → all tasks including closed
```

### Task CRUD

```
GET    /task/{taskId}
POST   /list/{listId}/task       body: {name, description, status, priority, due_date, start_date}
PUT    /task/{taskId}            body: same fields (partial update)
DELETE /task/{taskId}
```

Priority values: `1`=urgent, `2`=high, `3`=normal, `4`=low

### Views CRUD

```
GET    /space/{spaceId}/view
GET    /folder/{folderId}/view
GET    /list/{listId}/view
GET    /view/{viewId}
POST   /space/{spaceId}/view     body: {name, type, grouping?, columns?, filters?}
POST   /folder/{folderId}/view
POST   /list/{listId}/view
PUT    /view/{viewId}            body: {name?, columns?, filters?, grouping?, sorting?}
DELETE /view/{viewId}
```

**View types**: `list`, `board`, `calendar`, `gantt`, `table`, `timeline`, `workload`, `activity`, `map`, `conversation`, `doc`, `embed`, `form`

**Deletable views**: only user-created views (IDs like `8cbypq9-XXXXX`). System views (`4-SPACEID-28`, `5-FOLDERID-28`, `6-LISTID-8`) return errors on DELETE — skip them.

### Columns update format

```json
{
  "columns": {
    "fields": [
      {"field": "assignee", "idx": 0, "hidden": false, "width": 160},
      {"field": "dueDate",  "idx": 1, "hidden": false, "width": 160},
      {"field": "priority", "idx": 2, "hidden": false, "width": 160},
      {"field": "startDate","idx": 3, "hidden": true,  "width": 160}
    ]
  }
}
```
