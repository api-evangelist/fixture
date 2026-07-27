---
name: fixture-idempotent-activity-ingestion
description: Push events from an external system into the Fixture activity graph exactly once, using external_id as the idempotency key.
api: fixture-api-v1
base_url: https://beta-api.fixture.app/api/v1
operations:
  - createActivity
  - listActivities
  - getActivity
generated: '2026-07-20'
method: generated
source: openapi/fixture-v1-openapi.json, https://fixture.app/docs/api-reference/activities, https://fixture.app/docs/api-reference/errors
---

# Fixture: idempotent activity ingestion

Fixture's value comes from a complete activity graph. This is the flow for feeding it from your own systems — billing events, product usage, support tickets — without duplicating rows on retry.

## Before you start

- Scope required: `activities:write` (which also grants `activities:read`).
- At least one CRM record must be linked. Every Activity attaches to one or more of Contact, Account, or Deal.

## Steps

1. **Mint a stable `external_id`** from your own system's natural key — an invoice number, ticket ID, or event ID. It must be stable across retries for the same logical event, and unique across different events.
2. **Ingest with `createActivity`** (`POST /api/v1/activities`). Send `type` (a dotted event name such as `billing.invoice_paid`), `title`, `occurred_at` as an ISO 8601 datetime, optional `content`, `external_id`, and at least one of `contact_id`, `account_id`, or `deal_id`.
3. **Read the status code, not just the body:**
   - `201` — the Activity was newly created.
   - `200` — this was an idempotent replay; the existing Activity is returned unchanged. Treat as success and do not retry.
   - `409` with `error.code = idempotency_conflict` — the `external_id` already exists but your fields differ from the original. Do **not** retry blindly. Either replay with the original field values, or mint a new `external_id` because this is genuinely a different event.
   - `404` — the linked Contact, Account, or Deal ID does not exist in this workspace.
4. **Verify or reconcile** with `listActivities`, which supports `sort=occurred_at` / `sort=-occurred_at` and cursor pagination, or fetch a single record with `getActivity`.

## Rules

- `external_id` is the only caller-controlled correlation field in the API — there is no generic metadata bag and no `Idempotency-Key` header. Idempotency is scoped to Activity ingestion only; other creates are not idempotent, so guard those with your own dedupe check via a list call first.
- Activities link polymorphically: each entry in `entities[]` carries `entity_type`, `entity_id`, and a `role`. Read that array rather than assuming a single parent.
- Page with `limit` (default 50, max 100) plus `cursor` from `pagination.next_cursor`; stop when `pagination.has_more` is `false`.
- On `429 rate_limit_exceeded`, honor `Retry-After`. For backfills, batch and pace under 100 requests per minute per key.
