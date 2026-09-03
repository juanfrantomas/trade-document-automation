# Data Model: Mobile Commercial Invoice Capture and Processing

## Domain types

| Type | Fields / values | Rules |
| --- | --- | --- |
| `SubmissionId` | Opaque generated identifier | Unique in repository; never derived from document content or exposed as a path. |
| `InvoiceImage` | image ID, filename extension, declared MIME type, detected content type, byte length, storage key | Must be non-empty, JPEG/PNG/HEIC, ≤10 MiB, and have agreeing accepted signals before storage/extraction. |
| `DocumentSubmission` | ID, image IDs, state, attempt number, timestamps, final result | Contains one or more images. It moves from `processing` to exactly one final result; rejected uploads do not create it. |
| `FieldValue` | field name, normalized value (if any), reliability condition, optional confidence | Reliability is `trusted`, `missing`, `ambiguous`, `low_confidence`, or `inconsistent`. Provider terms are translated at the adapter boundary. |
| `ExtractedInvoice` | invoice number, date, seller, buyer, currency, invoice total, line items, optional shipment/reference fields | A provider-neutral result from all images belonging to the submission. |
| `LineItem` | description, quantity (optional), unit price (optional), line total | Where line items are present and totals can be evaluated, their aggregate must be consistent with invoice total for approval. |
| `ValidationFinding` | code, affected field/rule, reason, condition, severity | Safe to return in review context; never carries raw image bytes or provider exception detail. |
| `ProcessingOutcome` | `approved`, `review_required`, `processing_failed` | Exactly one final outcome for every accepted submission. |
| `SubmissionResult` | outcome, extracted safe summary, findings, safe next actions, completion timestamp | Immutable once final; used by retrieval and traceability. |

## State and outcome rules

```text
Unselected / invalid upload ──> upload_validation_failed (no submission, no outcome)
accepted upload ──> processing ──> approved
                              ├─> review_required
                              └─> processing_failed

processing_failed ──retry same submission──> processing
```

- `approved`: trusted invoice number, date, seller/exporter, buyer/importer, currency, and invoice
  total exist; available line-item totals are consistent; no applicable rule fails.
- `review_required`: extraction completed sufficiently to evaluate, but a required field/rule is
  missing, ambiguous, low confidence, inconsistent, or otherwise untrusted.
- `processing_failed`: extraction, temporary storage, or another technical pipeline step could not
  complete reliably. It is never used for a reviewable business-rule finding.
- An optional shipment/reference field does not prevent approval unless a future applicable rule
  makes it required.

## Relationships and ownership

One `DocumentSubmission` owns one or more `InvoiceImage` records and one current processing
attempt. An attempt receives at most one extraction result and produces one immutable
`SubmissionResult`. A result owns zero or more `ValidationFinding` values. `DocumentStorage` owns
temporary bytes; the repository owns safe metadata and result state.

## Persistence boundary

Feature 001 requires temporary in-process persistence to satisfy reconnect/retry behavior, but no
production database. The repository and document-storage ports are the only domain-facing storage
dependencies. A later durable adapter must preserve opaque IDs, atomic finalization, and safe
result retrieval without changing domain validation.
