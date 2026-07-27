---
name: fixture-contact-sync-with-merge-handling
description: Keep an external system's people and companies in sync with Fixture Accounts and Contacts, correctly following merged-Contact 301 and 410 responses.
api: fixture-api-v1
base_url: https://beta-api.fixture.app/api/v1
operations:
  - listContacts
  - getContact
  - createContact
  - updateContact
  - deleteContact
  - listAccounts
  - createAccount
  - getAccount
  - updateAccount
  - createNote
generated: '2026-07-20'
method: generated
source: openapi/fixture-v1-openapi.json, https://fixture.app/docs/api-reference/contacts, https://fixture.app/docs/api-reference/accounts
---

# Fixture: contact sync with merge handling

Fixture deduplicates people on its own side. Any long-running sync must handle a Contact ID it holds becoming a *pointer* to a surviving record.

## Before you start

- Scopes: `contacts:write` and `accounts:write` (each implies the matching read).
- Store Fixture IDs alongside your own records. They are opaque prefixed strings (`contact_...`, `account_...`) — never parse or reconstruct them.

## Steps

1. **Resolve or create the Account first.** Call `listAccounts` with date filters and cursor paging; if there is no match, call `createAccount` with `name`, `website_url`, `industry`, `employee_count`, and headquarters fields as available. A Contact's `account_id` should point at a real Account.
2. **Resolve or create the Contact.** `listContacts` to search your window, then `createContact` with `name`, `email`, `phone`, `title`, `account_id`, `linkedin_url`, `twitter_handle`. Note that a Contact exposes both a primary `email` and an `emails` array.
3. **Refresh a known Contact** with `getContact` and handle the merge signals:
   - `200` — normal, use `data`.
   - `301` — this Contact was merged. The body carries `merged_into`. **Repoint your stored ID at `merged_into` and re-read.**
   - `404` — the ID is not in this workspace.
4. **Write updates** with `updateContact` (PATCH), sending only changed fields. If you get `410`, the Contact is gone after a merge; the body still carries `merged_into` — retarget the write at the surviving Contact and retry once.
5. **Attach context** with `createNote`, linking the Note to the Account, Contact, or Deal it belongs to. Notes are create-only over the v1 API.
6. **Track freshness** using `last_activity_at`, `created_at`, and `updated_at`, plus the `updated_after` / `updated_before` list filters for incremental syncs.

## Rules

- Treat `301` and `410` as *redirects*, not failures. Both carry `merged_into`. Persist the new ID so you only follow the pointer once.
- Do not retry a `410` against the same ID — it will never succeed.
- `deleteContact` is destructive and returns `{"deleted": true, "id": "contact_..."}`.
- Branch on `error.code` (`not_found`, `forbidden`, `validation_error`), never on `error.message`.
- Incremental sync beats full sync: 100 requests per minute per API key, `limit` max 100 per page.
