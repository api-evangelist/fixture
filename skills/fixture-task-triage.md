---
name: fixture-task-triage
description: Create, assign, and complete Fixture Tasks against Accounts, Contacts, and Deals using the workspace's configurable Task Status vocabulary.
api: fixture-api-v1
base_url: https://beta-api.fixture.app/api/v1
operations:
  - listTaskStatuses
  - listTasks
  - getTask
  - createTask
  - updateTask
  - deleteTask
  - listUsers
generated: '2026-07-20'
method: generated
source: openapi/fixture-v1-openapi.json, https://fixture.app/docs/api-reference/tasks, https://fixture.app/docs/api-reference/task-statuses
---

# Fixture: task triage

Fixture treats Tasks as one queue shared by people and agents. Every commitment becomes a tracked Task with a link back to the record it belongs to.

## Before you start

- Scopes: `tasks:write` for the OAuth vocabulary, or the equivalent API key scope. `users:read` to resolve assignees.

## Steps

1. **Load the workspace vocabulary first.** `listTaskStatuses` (`GET /api/v1/task-statuses`) returns the Statuses configured for this workspace. This endpoint is small reference data — it returns only `data`, with **no** `pagination` object. Status names are workspace-configurable, so resolve them at runtime rather than hard-coding "Done".
2. **Resolve assignees** with `listUsers`, matching on `email` or `name` to get a user ID.
3. **Create the Task** with `createTask`, supplying `title`, `description`, `due_date`, `assignee_id`, the Status, and the polymorphic link `entity_type` + `entity_id` pointing at an Account, Contact, or Deal.
4. **Triage the queue** with `listTasks`. Filter with `created_after` / `updated_after`, page on `pagination.next_cursor`, and sort with `created_at` or `-created_at`. Compute overdue by comparing `due_date` against now, and treat a non-null `completed_at` as done.
5. **Move a Task** with `updateTask` (PATCH) — change the status, reassign via `assignee_id`, or push `due_date`. Re-read with `getTask` to confirm `task_status` and `task_status_id`.
6. **Remove only when asked.** `deleteTask` is destructive and returns `{"deleted": true, "id": "task_..."}`.

## Rules

- Never invent a Status name. Only use values returned by `listTaskStatuses`; a bad value returns `400 validation_error`.
- `created_by_id` and `assignee_id` are distinct — an agent-created Task assigned to a human has different values in each.
- The polymorphic parent is `entity_type` + `entity_id` together. Both are required to link a Task.
- Branch on `error.code`; back off on `429 rate_limit_exceeded` using `Retry-After`.
