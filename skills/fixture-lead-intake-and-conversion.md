---
name: fixture-lead-intake-and-conversion
description: Capture an inbound lead in Fixture and convert it into an Account, Contact, and optionally a Deal, handling the compound scope requirement.
api: fixture-api-v1
base_url: https://beta-api.fixture.app/api/v1
operations:
  - createLead
  - listLeads
  - getLead
  - updateLead
  - convertLead
generated: '2026-07-20'
method: generated
source: openapi/fixture-v1-openapi.json, https://fixture.app/docs/api-reference/leads, https://fixture.app/docs/authentication/scopes
---

# Fixture: lead intake and conversion

Turn raw inbound interest into qualified CRM records without creating duplicates.

## Before you start

- Authenticate with `Authorization: Bearer fx_...` (API key) or `Bearer eyJ...` (Fixture OAuth access token). Session cookies are rejected on `/api/v1/*`.
- Conversion needs **all** of `leads:write`, `accounts:write`, `contacts:write`, and additionally `deals:write` when you set `create_deal: true`. A missing scope returns `403` with `error.code = forbidden` and a message naming the scope. Check scopes before you start the flow, not after.
- Write implies read — `leads:write` already covers `leads:read`.

## Steps

1. **Check for an existing Lead.** Call `listLeads` with `limit` and the `created_after` / `updated_after` date filters to narrow the window. Sort with `sort=-created_at` (the default) or `sort=created_at`. Page with `cursor` from `pagination.next_cursor` until `pagination.has_more` is `false`.
2. **Create the Lead** with `createLead` if none matches. Leads carry denormalized company fields (`company_name`, `company_website`, `company_industry`, `company_employee_count`) alongside person fields (`name`, `email`, `phone`, `title`) plus `source`, `status`, and `estimated_value`. A `201` returns the Lead under `data`.
3. **Enrich as you learn more** with `updateLead` (PATCH). Send only the fields that changed.
4. **Read before converting** with `getLead`. If `converted_at` is already set, the Lead has been converted — reuse `converted_account_id` and `converted_contact_id` instead of converting again.
5. **Convert** with `convertLead` (`POST /api/v1/leads/{lead_id}/convert`). Set `create_deal: true` only when there is a real opportunity, and only when the token carries `deals:write`.
6. **Record the result.** The conversion response reports the resulting Account, Contact, and Deal. Store those opaque IDs (`account_...`, `contact_...`, `deal_...`) on your side and stop referencing the Lead ID for ongoing work.

## Rules

- IDs are opaque prefixed strings. Never parse or construct them.
- Branch on `error.code`, never on `error.message` — the message is human-readable and may change.
- Respect the limit of 100 requests per minute per API key. Watch `X-RateLimit-Remaining`, and on `429` (`rate_limit_exceeded`) wait `Retry-After` seconds and back off exponentially.
- `400 invalid_parameter` on a list call usually means a bad ISO 8601 date filter or an unsupported `sort` field. Lead lists support only `created_at` and `-created_at`.
