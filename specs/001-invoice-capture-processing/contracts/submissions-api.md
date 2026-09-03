# Submission API Contract

**Status**: Shared contract for Feature 001. Stabilize this document before frontend/backend work
is split into separate worktrees.

**Base path**: `/v1`

## Common types

`outcome` is one of `approved`, `review_required`, or `processing_failed`.
`state` is `processing` or `completed`. `outcome` is absent while state is `processing`.

Every safe error has this shape:

```json
{
  "error": {
    "code": "upload_validation_failed",
    "message": "One or more images cannot be accepted.",
    "details": [{"file": "invoice.heic", "code": "file_too_large", "message": "Maximum size is 10 MiB."}]
  },
  "request_id": "opaque-request-id"
}
```

`details` is optional. Messages and codes never contain stack traces, provider exception text, raw
invoice text, or image bytes.

## Create submission

`POST /v1/submissions`

Content type: `multipart/form-data`; one or more parts named `files`.

Each file must be a non-empty JPEG, PNG, or HEIC image, no larger than 10 MiB. The service checks
filename extension, supplied content type, and detected byte signature/content type. A whole
submission is rejected if any supplied file fails validation; extraction does not start.

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
| 400 | `missing_files` | No image files were supplied. |
| 413 | `file_too_large` | At least one image exceeds 10 MiB. |
| 415 | `unsupported_media_type` | Extension, declared MIME type, or detected type is not JPEG/PNG/HEIC. |
| 422 | `upload_validation_failed` | File is empty or accepted type signals do not agree. |

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
    "line_items": []
  },
  "findings": [
    {"code": "required_field_missing", "field": "buyer", "reason": "Buyer/importer is required for approval.", "condition": "missing"}
  ],
  "next_actions": ["human_review"]
}
```

For `approved`, `findings` is empty and `next_actions` is empty. For `processing_failed`, the
response contains a safe pipeline failure finding and `next_actions` includes `retry_submission`
and `replace_images`. `extracted_invoice` is omitted when extraction produced no safe structured
data. Values shown are synthetic examples only.

### Retrieval errors

| Status | `error.code` | Meaning |
| --- | --- | --- |
| 404 | `submission_not_found` | The opaque reference is unknown or has expired from temporary storage. |
| 500 | `internal_error` | Unexpected safe server failure; no internal detail is exposed. |

## Retry submission

`POST /v1/submissions/{submission_id}/retry`

Retries the temporarily retained accepted images for a completed `processing_failed` submission.
It returns the same `202 Accepted` processing response and retains the same submission reference.

| Status | `error.code` | Meaning |
| --- | --- | --- |
| 409 | `retry_not_allowed` | Submission is processing or did not finish as `processing_failed`. |
| 404 | `submission_not_found` | The reference or temporary images are unavailable. |

To replace images, the client starts a new `POST /v1/submissions` request; it does not alter a
completed submission.
