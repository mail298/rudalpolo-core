---
name: todoist
description: "Manage Todoist tasks and projects via REST API."
version: 1.0.0
author: Rudalpolo
license: MIT
platforms: [linux, macos, windows]
prerequisites:
  env_vars: [TODOIST_API_TOKEN]
metadata:
  hermes:
    tags: [Todoist, Task-Manager, Productivity, ToDo, API]
---

# Todoist API Integration

Manage tasks, projects, and due dates on Todoist via the official REST API v1.

## Authentication

Requires `TODOIST_API_TOKEN` in `${HERMES_HOME:-~/.hermes}/.env`.
Headers:
- `Authorization: Bearer $TODOIST_API_TOKEN`
- `Content-Type: application/json`

Base URL: `https://api.todoist.com/api/v1`

## Core Operations

### 1. List Active Tasks
```bash
curl -s -X GET "https://api.todoist.com/api/v1/tasks" \
  -H "Authorization: Bearer $TODOIST_API_TOKEN"
```

Filter by project:
```bash
curl -s -X GET "https://api.todoist.com/api/v1/tasks?project_id={project_id}" \
  -H "Authorization: Bearer $TODOIST_API_TOKEN"
```

### 2. Create a Task
```bash
curl -s -X POST "https://api.todoist.com/api/v1/tasks" \
  -H "Authorization: Bearer $TODOIST_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Deploy shortlink v2",
    "description": "Optional details",
    "due_string": "tomorrow at 10am",
    "priority": 4
  }'
```
*Note on priority:* In Todoist API: 4 = Urgent (P1), 3 = High (P2), 2 = Medium (P3), 1 = Normal (P4).

### 3. Complete / Close a Task
```bash
curl -s -X POST "https://api.todoist.com/api/v1/tasks/{task_id}/close" \
  -H "Authorization: Bearer $TODOIST_API_TOKEN"
```

### 4. Delete a Task
```bash
curl -s -X DELETE "https://api.todoist.com/api/v1/tasks/{task_id}" \
  -H "Authorization: Bearer $TODOIST_API_TOKEN"
```

### 5. List Projects
```bash
curl -s -X GET "https://api.todoist.com/api/v1/projects" \
  -H "Authorization: Bearer $TODOIST_API_TOKEN"
```

### 6. Create a Project
```bash
curl -s -X POST "https://api.todoist.com/api/v1/projects" \
  -H "Authorization: Bearer $TODOIST_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "DevOps Infrastructure"}'
```
