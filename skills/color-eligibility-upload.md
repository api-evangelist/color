---
name: Upload a Color eligibility file
description: >-
  Upload a CSV eligibility file to Color for one population, so covered members
  and their dependents become eligible to activate kits and enroll in Color
  programs — without accidentally invalidating the people already on the list.
api: openapi/color-external-api-v1-openapi.yml
operations:
- populations_eligibility_list
---

# Upload a Color eligibility file

`POST /populations/eligibility_list` on base `https://api.color.com/api/v1/external`.

## Prerequisites
- A Color-issued **bearer token** for the target environment. Staging and
  production have separate tokens and separate hosts:
  - Production: `https://api.color.com/api/v1/external`
  - Staging: `https://api.staging.color.com/api/v1/external`
- The `population` name Color assigned to your group. There is no endpoint that
  lists populations — you must be told the name out of band.
- A CSV eligibility file. Color's column guide and a sample file are linked from
  `https://docs.color.com/docs/csv-eligibility-file`. ANSI X12 834 is also
  accepted for file-based delivery (see `https://docs.color.com/docs/834-eligibility-file`).

## Steps
1. Set `Authorization: Bearer <token>` for the environment you are calling.
2. Call **`populations_eligibility_list`** as `multipart/form-data` with:
   - `eligibility_file` (required) — the CSV, as a file part.
   - `population` (required) — the population name. One request per population.
   - `replace` (optional boolean) — see the warning below. Defaults to `false`.
3. Read the `200` body, which is an `EligibilityListResponse`:
   - `results.created` / `results.updated` / `results.invalidated` — row counts.
   - `total` — active eligible entries in the population after the upload.
   - `ignored_columns` — columns Color did not recognise. **Always check this.**
     A silently ignored column is how a testing-cadence or external-id mapping
     goes missing.

## Rules and error handling
- Run against **staging first**, with the staging token. Staging contains only
  fake data.
- `replace: true` is **destructive**: every eligible participant not present in
  the uploaded file is invalidated. Use it only for a full refresh, and only when
  the file is known-complete. `replace: false` appends and updates in place.
- **This operation is not idempotent and accepts no idempotency key.** Do not
  blind-retry on `500`. Re-read the population with
  `eligibility_entries_list` and compare `total` before uploading again — a
  retried `replace: true` after a partial failure can invalidate people who
  should have stayed eligible.
- Errors: `400` = malformed request or invalid file (the body carries a list of
  error messages, unschema'd); `403` = token missing or not accepted — note that
  Color returns **403, not 401**, for auth failures; `500` = internal error.
- No rate limits are published and no rate-limit headers are returned. Pace
  uploads conservatively.
- For large or scheduled transfers, Color also supports SFTP
  (`transfer.color.com`, port 22, SSH key) with PGP encryption.
