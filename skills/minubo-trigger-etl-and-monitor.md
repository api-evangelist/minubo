---
name: Trigger a Minubo ETL run and monitor it
description: Authenticate, start an ETL process, and poll its status until the data load finishes.
api: openapi/minubo-api-openapi-original.json
operations: [auth_authenticateToken, etl_processStart, etl_getProcessStatus, etl_getProcessStatus_1, etl_getCurrentEtlDayStatus]
---

# Trigger a Minubo ETL run and monitor it

Use this skill to kick off a data load (extract-transform-load) for a Minubo tenant and
watch it to completion. Base URL: `https://api.minubo.com`.

## Steps

1. **Authenticate** — `auth_authenticateToken`: `POST /auth/v1/token` with
   `{"tokenId": "...", "tokenSecret": "..."}`; use the returned `token` as
   `Authorization: Bearer <token>` (valid 20 minutes).

2. **Start the ETL process** — `etl_processStart`: `POST /etl/v1/process/start`. Optional
   query param `forceFullLoad=true` forces a full reload (default incremental). A `202`
   returns `{"processUuid": "..."}`.

3. **Poll process status** — `etl_getProcessStatus`: `GET /etl/v1/process/status/{processUuid}`
   using the `processUuid` from step 2. Read `statusCode`; terminal states are `finished`,
   `error`, `aborted`, `killed`. Inspect `steps[]` (extract/transform/load) and `logs[]` for
   detail. `followupProcessUuid` links a chained follow-up run. Alternatively call
   `etl_getProcessStatus_1` (`GET /etl/v1/process/status/latest`) for the most recent run.

4. **Check the day schedule (optional)** — `etl_getCurrentEtlDayStatus`:
   `GET /etl/v1/status/current` for today's ETL day (`statusCode` of done/ongoing/
   not_started/on_hold/no_daily_update, plus a target ETA). A `404` here means no ETL is
   scheduled today.

## Rules (from conventions/ + errors/)

- Poll politely; do not tight-loop. `401` → re-authenticate (20-minute token TTL).
- `404` on a process UUID → the process does not exist; on the current day → nothing scheduled.
- No client-supplied idempotency key exists — avoid firing duplicate `process/start` calls;
  correlate a run by its returned `processUuid`.
- `500` → retry later.
