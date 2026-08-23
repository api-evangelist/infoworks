---
name: infoworks-onboard-source
description: Register a data source in Infoworks, browse and add its tables, crawl metadata, and run an ingestion job — with the rehearsal and rollback steps that this API's semantics require.
api: Infoworks REST API v3
base_url: "{protocol}://{host}:{port}/v3"
operations:
  - GET /sources
  - POST /sources
  - GET /sources/{source_id}
  - GET /sources/{source_id}/source_tables
  - POST /sources/{source_id}/fetch-tables
  - POST /sources/{source_id}/tables/source_tables
  - GET /sources/{source_id}/tables
  - GET /sources/{source_id}/config-migration
  - POST /sources/{source_id}/jobs
  - GET /admin/jobs/{job_id}
  - GET /admin/jobs/{job_id}/logs
  - GET /prodops/jobs/{job_id}/cancel
generated: '2026-08-23'
method: generated
source: openapi/infoworks-rest-api-v3-openapi.yml
---

# Onboard a source and ingest its tables

Authenticate first — see `infoworks-authenticate`.

## 1. Find or create the source

- `GET /v3/sources` (`getSourceList`) — paginate with `offset`/`limit`/`filter`. Check whether the
  source already exists before creating a duplicate; **there is no idempotency key on this API**, so a
  retried `POST /v3/sources` (`createSource`) creates a second source.
- `POST /v3/sources` (`createSource`) with the source config (name, type e.g. `rdbms`, sub_type e.g.
  `oracle`, `data_lake_path`, `environment_id`, `storage_id`).
- `GET /v3/sources/{source_id}` (`getSourceInfo`) to confirm.

## 2. Rehearse before you commit

This API has no dry-run mode, but it does have read-only discovery. Use it.

- `POST /v3/sources/{source_id}/fetch-tables` (`fetchTablesList`) — lists the tables visible in the
  source **without adding them**. This is the rehearsal step.
- `GET /v3/sources/{source_id}/source_tables` (`browseSource`) — browse the source structure.

## 3. Take the backup that is your only rollback

`GET /v3/sources/{source_id}/config-migration` (`exportSourceConfiguration`) exports the source
configuration. **Do this before any destructive or large change.** Every `DELETE` on this API is
permanent: there is no undelete, no trash, and no stated retention window anywhere in the docs. The
config export is the only recovery path Infoworks publishes.

## 4. Add tables and crawl metadata

`POST /v3/sources/{source_id}/tables/source_tables` (`getSourceTablesList`) adds the selected tables
to the source **and crawls their metadata**. The operationId reads like a getter; it is not — it
writes. Confirm with `GET /v3/sources/{source_id}/tables` (`getTablesList`).

## 5. Run ingestion

`POST /v3/sources/{source_id}/jobs` (`createJobForSource`) starts jobs for the source and its
artifacts. This spends real compute in the customer's cloud.

- **Do not retry a timeout.** There is no replay key. If the call times out, poll
  `GET /v3/admin/jobs` (`getJobs`, filter by source) to find out whether a job actually started before
  you consider issuing a second one.
- Poll `GET /v3/admin/jobs/{job_id}` (`getjobs id details`) for status.
- On failure read `GET /v3/admin/jobs/{job_id}/logs` (`get_job_logs_admin`) — it streams text.
- To stop a running job: `GET /v3/prodops/jobs/{job_id}/cancel` (`cancelJobByIdForProdOps`). Note this
  destructive action is exposed as a **GET**, so never speculatively prefetch job URLs.

## Error handling

| iw_code | HTTP | Meaning | Do |
|---|---|---|---|
| IW10006 | 400 | Invalid body or parameters | Fix the request; retrying unchanged will not help |
| IW10004 | 401/403 | Token expired or no access | Re-mint the JWT, retry once |
| IW10005 | 401 | Credential did not resolve | Stop; the credential is wrong |
| IW10002 | 500 | Server-side failure | Back off and retry; then pull the deployment logs |

Ignore the `help` URL in the error body — `api.infoworks.io` does not resolve.
