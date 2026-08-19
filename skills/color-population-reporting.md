---
name: Pull participant, sample and result data for a Color population
description: >-
  Read the four population reporting endpoints — participants, samples, results
  and self-reported results — page through them correctly, and join them back to
  your own member records. Every one of these returns identified patient data.
api: openapi/color-external-api-v1-openapi.yml
operations:
- populations_participants_list
- populations_samples_list
- populations_samples_read
- populations_results_list
- populations_self_reported_results_list
---

# Pull participant, sample and result data for a Color population

Base: `https://api.color.com/api/v1/external`, `Authorization: Bearer <token>`.

> **This is PHI.** Every operation here returns identified patient data — names,
> dates of birth, sex, email, phone, address, race/ethnicity, vaccination status
> and clinical results. Treat the whole surface as HIPAA-covered: pull the
> minimum necessary, do not log payloads, and do not hand raw responses to a
> model or tool that is not inside your covered boundary.

## Pagination — page number, not cursor
All four use `page` (1-based) and `page_size` (1–500, default 50), and return
`next` / `previous` / `results`. Loop until `next` is `null`.

This differs from `/eligibility/entries`, which is cursor-paginated. Branch per
endpoint.

## `populations_participants_list` — who is enrolled and where they stand
`GET /populations/participants`

- Filters: `populations`, plus the activation window
  `latest_sample_activated_at_start` / `latest_sample_activated_at_end`.
- Key field: `compliance_state` — `untested`, `compliant`,
  `activated_not_accessioned`, `overdue`, `positive`. This is enumerated in prose
  in the field description, not as a schema `enum`, so validate defensively.
- `samples[]` is embedded: the participant's activated samples, most recent
  first.
- `eligible_email` is the email from your eligibility file; `email` is the
  account email. They can differ — join on `external_id` or `participant_id`,
  not on `email`.

## `populations_samples_list` / `populations_samples_read` — kit lifecycle
`GET /populations/samples` and `GET /populations/samples/{kit_barcode}`

- `SampleApi` carries the timeline: `activated_at`, `received_at_lab`,
  `collected_at`, `resulted_at`, `cancelled_at`, plus `processing_lab`.
- `participant_id` on a sample is documented as the key to query participants —
  it is the join back to `populations_participants_list`.
- `404` on read means the barcode is not in Color's database.

## `populations_results_list` — released clinical results
`GET /populations/results`

- `significance`: `detected`, `not_detected`, `inconclusive`, `failed`.
- `is_revision: true` means this report supersedes an earlier one for the same
  sample. **Handle revisions** — treating them as new results double-counts.
- `released_at` is when the patient could see it; `opened_at` is when they did
  (null if never).
- `patient` and `sample` are embedded objects; `cycle_thresholds[]` carries
  per-gene PCR detail.

## `populations_self_reported_results_list` — results the member entered
`GET /populations/self_reported_results`

- Separate from lab results and lower trust — these are participant-entered.
- Filters: `reported_at_start`/`reported_at_end` and
  `test_date_start`/`test_date_end`. Note `test_date` is when the test was taken
  and `reported_at` is when it was reported; they are often far apart, so pick
  the window that matches your question.
- `ordering`: `reported_at` or `-reported_at`.

## Rules
- There is **no by-id read** for a participant, a result or a self-reported
  result. Everything except a sample is list-only — filter, do not fetch.
- There is **no population resource**. `population` is a bare string you must
  already know.
- `403` = auth failure (not 401). `500` = retry is safe; these are all reads.
- No rate limits and no rate-limit headers are published. `page_size: 500` is the
  only stated ceiling — pull large windows in fewer, bigger pages rather than
  hammering with small ones.
- For scheduled bulk delivery, prefer Color's SFTP CSV feeds (enrollments, risk,
  at-home testing, scheduling assistance, referrals) over polling these
  endpoints — see `https://docs.color.com/docs/receiving-color-data`.
