---
name: Upload a Color eligibility list
description: >-
  Upload an eligibility file (CSV or 834) to Color so covered members and their
  dependents become eligible to activate kits and enroll in Color programs.
api: openapi/color-eligibility-openapi.yml
operations:
- uploadEligibilityList
---

# Upload a Color eligibility list

Use this skill to enroll a population's members with Color by uploading an
eligibility file through the Eligibility List API.

## Prerequisites
- A Color-issued **bearer API token** for the target environment. Staging and
  production have **separate tokens and base hosts**:
  - Production: `https://api.color.com`
  - Staging (test): `https://external.staging.color.com`
- The `population` identifier Color assigned to your group.
- An eligibility file in a supported format (CSV or ANSI 834). See the CSV and
  834 file guides in Color's docs for the required columns.

## Steps
1. Choose your environment and its base host, and set the header
   `Authorization: Bearer <token>` with that environment's token.
2. Call **`uploadEligibilityList`** — `POST /api/v1/populations/eligibility_list`
   as `multipart/form-data` with:
   - `population` (required) — your population identifier.
   - `eligibility_file` (required) — the CSV or 834 file, as a file part.
   - `replace` (optional boolean) — `true` to fully replace the population's
     current eligibility list, otherwise the upload is treated as an update.
3. A `200` response means the file was accepted for processing. Members in the
   file then become eligible to activate Color kits and enroll in programs.

## Rules and error handling
- Do the first run against **staging** with the staging token before touching
  production.
- Setting `replace: true` is destructive — it replaces the whole eligibility
  list for the population; only use it for a full refresh.
- Handle responses: `400` = malformed request/invalid file, `401` = missing or
  invalid token, `403` = token not authorized for that population.
- No idempotency-key contract is documented, so avoid blind retries on `5xx`;
  confirm state before re-uploading to prevent duplicate processing.
- For very large or scheduled transfers, Color also supports SFTP
  (`transfer.color.com`, port 22, SSH key) with PGP encryption as an
  alternative to this API.
