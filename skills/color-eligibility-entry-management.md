---
name: Manage Color eligibility entries one at a time
description: >-
  Query, create and update individual eligibility entries in a Color population
  through the Eligibility V2 endpoints, instead of re-uploading a whole file —
  including how to invalidate a participant correctly.
api: openapi/color-external-api-v1-openapi.yml
operations:
- eligibility_entries_list
- eligibility_entries_create
- eligibility_entries_read
- eligibility_entries_update
---

# Manage Color eligibility entries one at a time

Base: `https://api.color.com/api/v1/external`. All four operations require
`Authorization: Bearer <token>`.

Use these instead of `populations_eligibility_list` when you need immediate,
programmatic changes to a handful of people. Use the file upload when you are
reconciling a whole population.

## Read the current list — `eligibility_entries_list`
`GET /eligibility/entries`. **Cursor-paginated**, not page-numbered.

- `cursor` — opaque; take it from the `next` URL, never construct it.
- `page_size` — 1–500, default 50.
- `ordering` — `created_at` or `-created_at` (default: newest first).
- `populations`, `unique_identifiers`, `external_ids` — comma-separated filters
  (`style: form`, `explode: false`). Omitting a filter means "all".

Loop until `next` is `null`. `next` and `previous` come back as fully-formed
URLs with the query parameters already applied.

> The `/populations/*` list endpoints paginate by **page number** instead. Do not
> reuse this loop for them.

## Create one — `eligibility_entries_create`
`POST /eligibility/entries` with an `EligibilityEntry` body. Required:
`unique_identifier`, `identifier_type`, `population`. Optional: `testing_cadence`,
`external_id`.

- `unique_identifier` + `identifier_type` is the join key. It must match the
  identifier you put on claims and member-event files, or Color cannot match the
  member.
- `external_id` is passed through untouched so you can reconcile results on your
  side.
- **Not idempotent, no idempotency key.** A retried POST creates a duplicate.
  Before retrying, call `eligibility_entries_list` filtered by
  `unique_identifiers` and check whether the entry landed.
- Success is `201` with the created entry, including its `id` (a UUID).

## Read one — `eligibility_entries_read`
`GET /eligibility/entries/{id}` where `id` is the UUID from the create/list
response.

## Update or invalidate — `eligibility_entries_update`
`PUT /eligibility/entries/{id}` with an `EligibilityEntryUpdate` body (same
fields, none required).

- There is **no DELETE**. To remove someone, set `invalidated_at` — invalidation
  is a field, not an operation.
- Typical updates: change `testing_cadence`, correct `external_id`, invalidate a
  departed employee.

## Error handling across all four
- `400` — malformed request; body is an unschema'd list of error messages.
- `403` — token missing or not accepted. Color uses **403, not 401**, and sends
  no `WWW-Authenticate` challenge.
- `404` — "Result could not be retrieved" on read, "Result does not exist" on
  update.
- `500` — retry is safe on the two read operations only.
- No `429` and no rate-limit headers are documented.
