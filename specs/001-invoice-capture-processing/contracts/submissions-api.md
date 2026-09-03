# Submission API Contract

**Status**: Locked for Feature 001. This is the shared frontend/backend contract; any change to
endpoints, field names, outcome values, validation codes, or HTTP behavior requires coordinated,
single-owner review before implementation work is split into separate worktrees.

**Base path**: `/v1`

## Common types

`submission_id` is an opaque, server-generated identifier. Clients must treat it as an uninterpreted
string and use it only in the supplied `status_url` and retry path; it contains no invoice content,
user identity, timestamp guarantee, or provider identifier.

`state` is one of `processing` or `completed`. `outcome` is absent while `state` is `processing`.
For `completed`, `outcome` is exactly one of `approved`, `review_required`, or
`processing_failed`.

Every safe error has this shape:

```json
{
  "error": {
    "code": "upload_validation_failed",
    "message": "One or more images cannot be accepted.",
    "details": [{"file_index": 0, "code": "file_too_large", "message": "Maximum size is 10 MiB."}]
  },
  "request_id": "opaque-request-id"
}
```

`details` is optional. A detail identifies a multipart file by zero-based `file_index`, never by
filename or image content. Messages and codes never contain stack traces, provider exception text,
raw invoice text, image bytes, secrets, or internal storage details.

`ValidationFinding` has `code`, `field` (a documented field name or `null` for a pipeline
condition), `reason`, and `condition`. `condition` is one of `trusted`, `missing`, `ambiguous`,
`low_confidence`, `inconsistent`, or `technical_failure`.

## Create submission

`POST /v1/submissions`

Content type: `multipart/form-data`; one or more file parts named `files`, in client-selected order.
There is no Feature 001 count limit. All supplied images form one commercial-invoice submission.

Each file must be non-empty and no larger than 10 MiB (10 × 1024 × 1024 bytes). Accepted formats
are JPEG (`.jpg` or `.jpeg`, `image/jpeg`), PNG (`.png`, `image/png`), and HEIC (`.heic`,
`image/heic`). The service checks filename extension, declared part content type, and detected byte
signature/content type. All three signals must identify the same accepted format. A whole
submission is rejected if any supplied file fails validation; no images are stored for processing
and extraction does not start.

### Accepted response — `202 Accepted`

```json
{
  "submission_id": "sub_opaque",
  "state": "processing",
  "status_url": "/v1/submissions/sub_opaque"
}
```

### Upload errors

| Status | `error.code` | Meaning |
| --- | --- | --- |
| 415 | `invalid_request_content_type` | The request is not `multipart/form-data`. |
| 400 | `missing_files` | No image files were supplied. |
| 413 | `file_too_large` | At least one image exceeds 10 MiB. |
| 415 | `unsupported_image_type` | At least one extension, declared MIME type, or detected type is not an accepted image format. |
| 422 | `upload_validation_failed` | A file is empty or its accepted type signals do not agree. |

These are file-acceptance outcomes, not `processing_failed` results.

## Retrieve submission

`GET /v1/submissions/{submission_id}`

### Processing response — `200 OK`

```json
{
  "submission_id": "sub_opaque",
  "state": "processing",
  "status_url": "/v1/submissions/sub_opaque"
}
```

### Completed response — `200 OK`

```json
{
  "submission_id": "sub_opaque",
  "state": "completed",
  "outcome": "review_required",
  "extracted_invoice": {
    "invoice_number": {"value": "SYN-1001", "condition": "trusted"},
    "invoice_date": {"value": "2026-09-01", "condition": "trusted"},
    "seller": {"value": "Synthetic Exporter", "condition": "trusted"},
    "buyer": {"value": null, "condition": "missing"},
    "currency": {"value": "EUR", "condition": "trusted"},
    "invoice_total": {"value": "120.00", "condition": "trusted"},
    "line_items": [
      {"quantity": "2", "unit_price": "60.00", "line_total": "120.00", "condition": "trusted"}
    ],
    "shipment_reference": {"value": null, "condition": "missing"}
  },
  "findings": [
    {"code": "required_field_missing", "field": "buyer", "reason": "Buyer/importer is required for approval.", "condition": "missing"}
  ],
  "next_actions": ["human_review"]
}
```

`extracted_invoice` is the safe structured summary. It has `invoice_number`, `invoice_date`,
`seller`, `buyer`, `currency`, `invoice_total`, `line_items`, and optional
`shipment_reference`. Every scalar field is a `{ "value": string | null, "condition": ... }`
object. Every line item contains available normalized numeric values and its condition; descriptions
and raw source text are not returned. Values shown are synthetic examples only.

For `approved`, `findings` and `next_actions` are empty. For `review_required`, `findings` includes
the affected field or rule, safe reason, and reliability/consistency condition; `next_actions` is
`["human_review"]`. For `processing_failed`, `findings` contains a safe pipeline finding with
`field: null` and `condition: "technical_failure"`; `next_actions` is
`["retry_submission", "replace_images"]`. `extracted_invoice` is omitted only when no safe
structured extraction data exists.

### Retrieval errors

| Status | `error.code` | Meaning |
| --- | --- | --- |
| 404 | `submission_not_found` | The opaque reference is unknown or has expired from temporary storage. |
| 500 | `internal_error` | Unexpected safe server failure; no internal detail is exposed. |

## Retry submission

`POST /v1/submissions/{submission_id}/retry`

Retries the temporarily retained accepted images for a completed `processing_failed` submission.
It requires no request body, creates a new processing attempt for the same `submission_id`, and
returns the same `202 Accepted` processing response. It never changes an approved or
review-required submission. Replacement images always use a new `POST /v1/submissions` request and
therefore receive a new opaque submission identifier.

| Status | `error.code` | Meaning |
| --- | --- | --- |
| 409 | `retry_not_allowed` | Submission is processing, approved, or review-required. |
| 404 | `submission_not_found` | The reference or temporary images are unavailable. |

## Contract invariants

- Upload-validation errors are HTTP errors and never create a processing outcome.
- Every accepted submission reaches exactly one final outcome per processing attempt.
- `review_required` is a completed, reviewable business-validation result; it is not a technical
  failure.
- `processing_failed` means a technical pipeline step could not reliably complete; it is never
  used for a reviewable business-rule finding.
- Clients poll the supplied `status_url` after a `202` response and may use the same reference
  after reconnecting while temporary Feature 001 state remains available.
