---
name: fixture-deal-pipeline-management
description: Create a Deal against an Account, attach the buying committee, and move it through Fixture's configurable Pipeline stages.
api: fixture-api-v1
base_url: https://beta-api.fixture.app/api/v1
operations:
  - listPipelines
  - getPipeline
  - listAccounts
  - createDeal
  - listDeals
  - getDeal
  - updateDeal
  - listDealContacts
  - updateDealContacts
  - listUsers
generated: '2026-07-20'
method: generated
source: openapi/fixture-v1-openapi.json, https://fixture.app/docs/api-reference/deals, https://fixture.app/docs/api-reference/pipelines
---

# Fixture: deal and pipeline management

Pipelines and Stages are configured in the Fixture UI and are **read-only** over the API. Your job is to place Deals into the right Stage, not to invent stages.

## Before you start

- Scopes: `deals:write` to create and move Deals, `pipelines:read` to resolve stages, `accounts:read` to resolve the parent Account, `users:read` to resolve an owner.

## Steps

1. **Resolve the Pipeline.** Call `listPipelines`, then `getPipeline` for the one you want. Each Pipeline embeds `stages[]`, and each stage carries `id`, `name`, `position`, `color`, and `stage_type`. Match on `name` or `stage_type` and keep the stage `id` — never hard-code a stage ID across workspaces.
2. **Resolve the Account** with `listAccounts` (or an ID you already hold). A Deal belongs to exactly one Account via `account_id`.
3. **Create the Deal** with `createDeal`. Supply `name`, `account_id`, `pipeline_id`, `pipeline_stage_id`, and where known `value` and `expected_close_date`. `201` returns the Deal under `data`.
4. **Attach the buying committee.** `updateDealContacts` (`PUT /api/v1/deals/{deal_id}/contacts`) sets the Contact associations — it replaces the set, so read the current set with `listDealContacts` first and send the merged result.
5. **Advance the Deal** with `updateDeal` (PATCH), changing `pipeline_stage_id`. Re-read with `getDeal` to confirm the derived `stage_name` and `stage_type` now reflect the new Stage.
6. **Review the book of business** with `listDeals`, using `updated_after` / `created_after` filters and `sort=-created_at`, paging on `pagination.next_cursor`.

## Rules

- `stage_name` and `stage_type` on a Deal are derived from `pipeline_stage_id`. Write the ID; read the names.
- Ownership lives under `relationships.owner` on Accounts and Deals. Resolve owner IDs with `listUsers` (`users:read`).
- PATCH is partial — send only changed fields.
- `deleteDeal` is destructive and returns `{"deleted": true, "id": "deal_..."}`. Confirm before calling it.
- Watch `X-RateLimit-Remaining`; back off on `429 rate_limit_exceeded`.
