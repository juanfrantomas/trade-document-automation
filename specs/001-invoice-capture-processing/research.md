# Research: Mobile Commercial Invoice Capture and Processing

## Decisions

### Use a provider-neutral extraction port with a deterministic local adapter

**Decision**: Define `DocumentExtractor` in the ports layer and start with a local fixture adapter
that maps hashes of synthetic image fixtures to deterministic extraction results.

**Rationale**: The domain receives stable, provider-neutral extracted fields and reliability
conditions. Tests remain repeatable without network access, cloud credentials, or real invoices.

**Alternatives considered**:

- Azure Document Intelligence: deferred by the constitution and feature scope.
- Tesseract or another live local OCR engine: adds operational variability before the first
  vertical slice proves the rules and API.
- Extraction logic in FastAPI handlers: rejected because it couples business processing to HTTP.

### Use temporary in-memory persistence and document storage

**Decision**: Implement `SubmissionRepository` and `DocumentStorage` ports with in-memory adapters
for this feature.

**Rationale**: A stored reference/result is necessary to resume after reconnecting and retry the
same submission. A production database is not justified for one-process, one-interaction scope;
ports preserve a direct path to durable adapters later.

**Alternatives considered**:

- Stateless synchronous-only processing: cannot meet reconnect/retry requirements safely.
- Production database/object storage: unnecessary operational complexity for Feature 001.
- Browser-only retry state: cannot safely retrieve a completed result after leaving the page.

### Use a two-step REST submission contract

**Decision**: `POST /v1/submissions` accepts multipart images and returns `202` with an opaque
submission reference and `processing` state; `GET /v1/submissions/{submission_id}` returns the
state, final outcome, safe extracted summary, and findings.

**Rationale**: It directly represents mobile processing progress and reconnect recovery while
allowing the local adapter to remain inline initially or become asynchronous later without a
breaking frontend contract.

**Alternatives considered**:

- One blocking upload endpoint: does not model the required in-progress/reconnect flow cleanly.
- WebSockets: unnecessary for a small polling-based vertical slice.
- Frontend-defined response shapes: rejected; the documented backend contract is the authority.

### Use React, TypeScript, and Vite for the first mobile web UI

**Decision**: Propose a minimal React + TypeScript + Vite frontend, with no design system or state
library unless a concrete need emerges.

**Rationale**: It offers a compact component model, simple API integration, and good testing and
portfolio ergonomics while remaining focused on one capture-to-result flow.

**Alternatives considered**:

- Frameworkless browser code: viable but less maintainable as result and retry states grow.
- Next.js: unnecessary server-rendering and framework surface for this feature.
- Mobile-native app: outside the mobile-browser requirement.

### Verify uploads through independent signals

**Decision**: Validate every image's extension, declared MIME type, size, non-empty content, and
detected content signature before extraction.

**Rationale**: Browser-provided MIME and filenames alone are untrusted. Cross-checking is required
by the constitution and keeps invalid uploads distinct from technical processing failures.

**Alternatives considered**:

- Extension-only validation: insecure and explicitly prohibited.
- MIME-only validation: browser-controlled and insufficient.
- Full antivirus scanning: deferred; it requires a separate public-upload threat model.
