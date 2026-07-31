---
name: Query Minubo commerce data
description: Authenticate, discover the tenant's queryable schema, and run a data query against Minubo's modeled commerce data.
api: openapi/minubo-api-openapi-original.json
operations: [auth_authenticateToken, data_getSchema, data_query]
---

# Query Minubo commerce data

Use this skill to pull modeled commerce metrics (orders, products, customers) from a
Minubo tenant via the Data API. Base URL: `https://api.minubo.com`.

## Steps

1. **Authenticate** — `auth_authenticateToken`: `POST /auth/v1/token` with a JSON body
   `{"tokenId": "...", "tokenSecret": "..."}` (credentials from the Minubo app under
   Settings -> API Credentials). Read `token` from the response and send it as
   `Authorization: Bearer <token>` on every subsequent call. The token is valid for
   20 minutes — re-authenticate on `401`.

2. **Discover the schema** — `data_getSchema`: `GET /data/v1/schema`. The response lists
   `attributes[]` and `measures[]`, each with a stable `code` plus localized `nameEN` /
   `nameDE`. Only use `code` values from this response — the schema reflects what your
   tenant is allowed to query.

3. **Run the query** — `data_query`: `POST /data/v1/query` with `attributes[]` (codes),
   `measures[]` (codes), a required `timeFilter` (`{from, to}` dates), optional
   `comparisonTimeFilter`, optional `filter[]` and `orderBy`, and a required `limit`.

## Rules (from conventions/ + errors/)

- Max 20 `attributes` and 20 `measures` per request; `limit` must be 0–200000.
- `timeFilter` is required and `from` must not be after `to` (same for `comparisonTimeFilter`).
- `IN` / `NOT_IN` filters require a non-empty `values` list; `EXISTS` / `NOT_EXISTS` must omit `values`.
- `orderBy.field` must reference a field named in `attributes` or `measures`.
- `403 Field not allowed` → the field code is not permitted for the tenant; re-check the schema.
- `422 Unprocessable query` → a constraint above was violated.
- `429` → the Data API allows 50 requests per 10 minutes per token; back off and retry.
- There is no idempotency-key contract and no cursor pagination — a single request returns up to `limit` rows.
