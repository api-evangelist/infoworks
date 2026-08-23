---
name: infoworks-run-workflow
description: Start, monitor, pause, resume and cancel an Infoworks orchestration workflow, read its runs, tasks, logs and lineage, and work with workflow versions.
api: Infoworks REST API v3
base_url: "{protocol}://{host}:{port}/v3"
operations:
  - GET /domains
  - GET /domains/{domain_id}/workflows
  - GET /domains/{domain_id}/workflows/{workflow_id}
  - POST /domains/{domain_id}/workflows/{workflow_id}/start
  - GET /domains/{domain_id}/workflows/{workflow_id}/runs
  - GET /domains/{domain_id}/workflows/{workflow_id}/runs/{workflow_run_id}
  - POST /domains/{domain_id}/workflows/{workflow_id}/runs/{workflow_run_id}/cancel
  - POST /domains/{domain_id}/workflows/{workflow_id}/runs/{workflow_run_id}/restart
  - POST /domains/{domain_id}/workflows/{workflow_id}/pause
  - POST /domains/{domain_id}/workflows/{workflow_id}/resume
  - GET /domains/{domain_id}/workflows/{workflow_id}/runs/{workflow_run_id}/tasks/{task_id}/logs
  - GET /domains/{domain_id}/workflows/{workflow_id}/lineage
  - GET /domains/{domain_id}/workflows/{workflow_id}/versions
  - GET /domains/{domain_id}/workflows/{workflow_id}/config-migration
generated: '2026-08-23'
method: generated
source: openapi/infoworks-rest-api-v3-openapi.yml
---

# Run and supervise an Infoworks workflow

Authenticate first — see `infoworks-authenticate`. Workflows live inside a **domain**, which is the
access-control boundary; you need a `domain_id` before anything else.

## 1. Locate the workflow

- `GET /v3/domains` (`getDomains`).
- `GET /v3/domains/{domain_id}/workflows` (`getDomainWorkflows`) — `offset`/`limit`/`filter` apply.
- `GET /v3/domains/{domain_id}/workflows/{workflow_id}` (`getWorkflow`) to read its configuration.
- `GET /v3/domains/{domain_id}/workflows/{workflow_id}/lineage` (`getWorkflowLineage`) and
  `/ancestors` / `/parents` if you need to know what this run will affect downstream **before** you
  start it. On an orchestration API, lineage is the closest thing to a blast-radius check.

## 2. Start it — once

`POST /v3/domains/{domain_id}/workflows/{workflow_id}/start` (`startWorkflow`) runs the latest version.

**This is the single most dangerous call on this API for an agent.** It is not idempotent, it takes no
idempotency key, and it spends real compute. If the request times out or the connection drops:

1. **Do not retry.**
2. `GET /v3/domains/{domain_id}/workflows/{workflow_id}/runs` (`getWorkflowRunsByWorkflowId`).
3. Only start again if no new run appeared.

## 3. Monitor

- `GET .../runs/{workflow_run_id}` (`getWorkflowRun`) for run status.
- `GET .../runs/{workflow_run_id}/tasks/{task_id}/logs` (`getWorkflowRunTaskLogsByAttempt`) for a
  failing task; `/logs/download` (`downloadWorkflowRunTaskLogs`) for the whole file.

## 4. Intervene

| Intent | Call | Notes |
|---|---|---|
| Stop this run | `POST .../runs/{workflow_run_id}/cancel` (`cancelWorkflowRun`) | Stops execution. Work already written to the target warehouse is **not** rolled back. |
| Re-drive this run | `POST .../runs/{workflow_run_id}/restart` (`restartWorkflowRun`) | Restart, not undo. |
| Hold all runs | `POST .../workflows/{workflow_id}/pause` (`pauseWorkflow`) | Pauses all running builds. |
| Release them | `POST .../workflows/{workflow_id}/resume` (`resumeWorkflow`) | |
| Stop many at once | `POST /v3/admin/workflows/cancel` (`cancelMultipleWorkflowRuns`) | Admin-scope, bulk. |
| Clear a stuck lock | `POST /v3/admin/unlock-entities/{entity_id}` (`unlockEntity`) | For entities left locked by a failed run. |

**No reversal window is published for any of these.** The docs state no deadline after which a cancel
stops working, and none should be assumed either way.

## 5. Versions and recovery

Workflow versioning arrived in product release 6.2.0.

- `GET .../workflows/{workflow_id}/versions` (`getWorkflowVersions`), `POST` to create
  (`createWorkflowVersion`), `POST .../versions/{workflow_version_id}/set-active` (`setActiveVersion`).
- `GET .../workflows/{workflow_id}/config-migration` (`getExportConfigForMigration`) exports the
  configuration; `PUT` imports it (`importConfigForMigration`). **`DELETE` on a workflow or a workflow
  version is permanent** — take the export first.

## Scheduling

`GET .../workflows/{workflow_id}/schedules` (`getWorkflowSchedule`), then
`PUT .../schedules/enable` (`setWorkflowSchedule`) or `PUT .../schedules/disable`
(`disableWorkflowSchedule`). Disabling a schedule is the reversible way to stop future runs; deleting
the workflow is not.
