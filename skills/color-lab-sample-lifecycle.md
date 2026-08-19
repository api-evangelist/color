---
name: Report lab sample events to Color (LIMS integration)
description: >-
  For a processing lab integrating its LIMS with Color — accession an arriving
  sample, report its result, or record that an unapproved sample was destroyed.
  These are the only three operations in Color's API documented as idempotent.
api: openapi/color-external-api-v1-openapi.yml
operations:
- samples_accession_create
- samples_results_create
- samples_destroy_create
---

# Report lab sample events to Color (LIMS integration)

Base: `https://api.color.com/api/v1/external`, `Authorization: Bearer <token>`.

These three operations are the lab-facing write surface. They move a sample
through its lifecycle in Color's system. Call them in order: **accession →
result**, or **accession → destroy** when the sample is not approved for
processing.

`sample_barcode` is a path parameter and Color publishes its format as a regex:
`^D-\d{10}$` (e.g. `D-1234567890`). Validate before calling — a malformed
barcode comes back as a `404`, indistinguishable from an unknown sample.

## 1. `samples_accession_create` — the sample arrived
`POST /samples/{sample_barcode}/accession`

- **Idempotent.** Color's own wording: "If the sample was previously accessioned
  by the caller, its status will not be modified."
- The `200` body carries `accession_number` (e.g. `C-12345`), `collected_at`, and
  `is_approved_for_processing`.
- **Branch on `is_approved_for_processing`.** If it is `false`, the sample must
  not be run — go to `samples_destroy_create`. `collected_at` may be `null` while
  it is false.
- `400` here is unusual and is **not** a retry case: it means the sample was
  already accessioned by a *different* lab, or Color hit an internal problem. The
  body carries an `error` property. Escalate rather than loop.

## 2. `samples_results_create` — report the result
`POST /samples/{sample_barcode}/results`

- **Idempotent on (assay, completion time).** "If the result has already been
  reported (identified by the assay and the completion time), nothing happens."
  Send the same `assay_type` + `time_completed` on a retry and you will not
  double-report.
- Required body fields: `significance`, `assay_type`, `time_completed`.
  Optional: `batch_id` (your plate ID — Color uses it for sample tracking and
  investigations; send it).
- `significance` is a closed vocabulary published in the spec, including `NEG`,
  `POS`, `INCONCLUSIVE`, `INVALID` (assay failed twice) and the
  `UNSATISFACTORY_1`…`UNSATISFACTORY_n` rejection reasons (wrong transport
  medium, bad collection timing, unlabeled tube, insufficient volume, tube data
  mismatch). Read the enum from the contract rather than hard-coding a subset.
- `400` means the sample is **not approved for processing** or **not
  accessioned** — the message names which. Fix the precondition; do not retry as-is.

## 3. `samples_destroy_create` — the sample was destroyed
`POST /samples/{sample_barcode}/destroy`

- **Idempotent.** "If the sample was already destroyed, nothing happens."
- Only for samples that are **not** approved for processing. A `400` means the
  sample *is* approved and must be run instead.
- This records a physical, irreversible action. Confirm
  `is_approved_for_processing: false` from the accession response before calling.

## Rules across all three
- `404` = barcode not in Color's database (or malformed).
- `500` = "An error has occurred, please try again after a few minutes." Because
  all three are idempotent, a bounded retry with backoff is safe here — this is
  the one part of Color's API where that is true.
- `403` = token missing or not accepted. Color returns 403, not 401.
- These three operations carry no `operationId` in Color's published spec; the
  identifiers used here are assigned in
  `overlays/color-external-api-v1-overlay.yaml` and are stable within this repo.
- No rate limits are published and no rate-limit headers are returned.
