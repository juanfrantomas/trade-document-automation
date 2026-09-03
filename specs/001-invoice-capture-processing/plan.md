# Implementation Plan: Mobile Commercial Invoice Capture and Processing

**Branch**: `001-invoice-capture-processing` | **Date**: 2026-09-03 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification and approved clarification session for Feature 001.

## Summary

Deliver a mobile-first vertical slice where a user uploads one or more commercial-invoice images,
receives a submission reference and processing state, and retrieves a safe final outcome. The
Python backend uses hexagonal architecture: upload validation and invoice business rules are pure
domain/application code; FastAPI, in-memory persistence and document storage, and a deterministic
local extractor are adapters. The local extractor is the only initial provider; Azure Document
Intelligence is intentionally excluded.

## Technical Context

**Language/Version**: Python 3.12

**Primary Dependencies**: FastAPI and Pydantic at HTTP/application boundaries; uvicorn for local
serving; pytest, httpx, ruff, and mypy for verification. Frontend proposal: React + TypeScript +
Vite, selected for a small, maintainable mobile UI and straightforward component testing.

**Storage**: No production database. An in-memory `SubmissionRepository` and temporary in-memory
`DocumentStorage` adapter retain a submission's safe result and retryable image bytes for the
running process only. Both are ports so durable storage can be added later without changing domain
or use-case logic.

**Testing**: pytest domain, use-case, extractor-contract, and FastAPI integration tests; frontend
component tests; Playwright mobile-flow tests once the UI exists.

**Target Platform**: Linux-hosted FastAPI service and modern mobile browsers; desktop remains
supported as a secondary experience.

**Project Type**: Monorepo web application (`backend/`, `frontend/`, `specs/`, `docs/`).

**Performance Goals**: 95% of accepted submissions expose a final result within 60 seconds under
normal operating conditions; local deterministic fixtures should complete well below that target.

**Constraints**: JPEG, PNG, and HEIC only; maximum 10 MiB per image; validate extension, declared
MIME type, and detected content type; any number of images is a single submission; only synthetic
fixtures; no secrets or raw invoice contents in logs or errors.

**Scale/Scope**: One invoice submission at a time per user interaction; no authentication, history,
bulk queue, production persistence, provider integration, or reviewer decision UI in Feature 001.

## Constitution Check

### Pre-design gate — PASS

| Principle | Plan evidence |
| --- | --- |
| Hexagonal architecture | Domain, application use cases, ports, and FastAPI/in-memory/local adapters are separate. |
| Vendor independence | `DocumentExtractor` is a port; the local adapter is replaceable without domain changes. |
| Specification-driven delivery | This plan derives from `spec.md`; `tasks.md` is deliberately not created here. |
| Practical TDD | Test layers cover domain rules, orchestration, adapter contract, HTTP integration, and later mobile flows. |
| Secure by design | Multi-signal upload validation, synthetic-only fixtures, safe errors, and metadata-only logging are specified. |
| Human oversight | Every accepted submission has exactly one final outcome; review context retains safe reasons. |
| Traceability | A submission reference, outcome, findings, and safe event metadata are retained by the repository port. |
| Mobile-first | The contract supports upload → processing → result/reconnect; the frontend remains intentionally small. |
| API before parallel work | `contracts/submissions-api.md` is the shared contract and is the required synchronization point. |
| Initial constraints | Python 3.12, uv, FastAPI, monorepo, local/mock extraction, and no Azure integration are honored. |

No constitution violations or exceptions are required.

## Architecture and Processing Design

### Backend boundaries

`domain` contains invoice values, field reliability, findings, outcome rules, and state invariants.
It imports neither FastAPI nor Pydantic. `application` owns the submit, process, retrieve, and
retry use cases and depends only on domain types and ports. `ports` define interfaces for document
extraction, submission persistence, document storage, safe clock/ID generation where needed, and
safe event recording. `adapters` implements those interfaces. `api` is the FastAPI inbound adapter
that parses multipart input into boundary models, invokes use cases, and maps domain/application
results to the documented HTTP contract.

Processing sequence:

1. FastAPI streams and validates every uploaded image: non-empty, ≤10 MiB, allowed extension,
   declared MIME type, and detected signature/content type must agree.
2. A valid submission receives a generated opaque reference, image bytes are stored temporarily,
   and a `processing` record is created. Upload rejection returns an upload-validation error and
   creates no processing outcome.
3. The processing use case reads the stored images and calls `DocumentExtractor`.
4. A technical extractor/storage failure that prevents reliable processing completes the submission
   as `processing_failed` with a safe reason.
5. A completed extraction becomes domain `ExtractedInvoice`; domain validation produces findings.
   Required missing, ambiguous, low-confidence, or inconsistent information produces
   `review_required`; only trusted required fields and consistent available line-item totals produce
   `approved`.
6. The repository stores one immutable final outcome, safe extracted summary, findings, and
   metadata-only events. `GET` exposes the state/result for reconnect. Retry creates a new
   processing attempt for the same submission rather than a duplicate submission reference.

The initial local adapter may complete inline while preserving the `processing` contract state.
No background worker, queue, database, or cloud document store is required for this vertical slice.

### Explicit ports

| Port | Purpose | Initial adapter |
| --- | --- | --- |
| `DocumentExtractor` | Convert stored invoice images into provider-neutral extracted fields and extraction status. | `LocalFixtureDocumentExtractor`, deterministically maps synthetic fixture content digests to predefined results. |
| `SubmissionRepository` | Create, retrieve, and atomically complete/retry submission records and safe outcomes. | Process-local in-memory repository. |
| `DocumentStorage` | Temporarily save/read retryable image bytes by opaque submission reference; delete on configured process/TTL cleanup. | Process-local in-memory storage. |
| `SafeEventRecorder` | Record status transitions and failures using references, codes, and durations only. | Structured application logger adapter. |

The extractor input/output contains provider-neutral bytes, image metadata, extracted values,
reliability conditions, and non-sensitive failure codes. Azure, Google, AWS, Tesseract, or another
provider is added as an adapter only; its SDK types never cross the port.

### API contract and collaboration boundary

The authoritative shared contract is [contracts/submissions-api.md](./contracts/submissions-api.md).
It must be reviewed and treated as stable before frontend/backend work is separated into worktrees.
Any change to response fields, outcome values, validation codes, or HTTP behavior is a coordinated
shared-file change by its single owner.

Safe later parallelization, after that contract is stable:

1. **Backend owner**: domain/application/ports/adapters and API integration tests.
2. **Frontend owner**: mobile capture/result UI using only the contract fixtures.
3. **Independent reviewer**: test coverage, architecture boundary, security, and contract review;
   does not edit files owned by either implementation stream.

The synchronization point is an approved contract plus fixture examples. Before that point, the
architecture, contract, and documentation remain serial work under one owner.

### Security and observability

Required now: enforce byte limits while reading uploads; verify extension, declared MIME type, and
detected content signature; reject empty/mismatched files before extraction; use opaque references;
return stable safe error codes/messages; keep raw images and extracted raw content out of logs;
use only synthetic fixtures; and load any runtime configuration from environment variables.

Deferred: authentication/authorization, tenant isolation, malware scanning, rate limiting, durable
encrypted storage, retention/deletion policy, audit export, cloud-provider credentials, and public
upload threat-model revisions. They are outside Feature 001 and must be planned before introducing
public or durable production service boundaries.

### Architecture documentation

Create one concise ADR, [docs/adr/0001-feature-001-hexagonal-processing-boundaries.md](../../docs/adr/0001-feature-001-hexagonal-processing-boundaries.md), covering the hexagonal boundaries,
extractor port, temporary storage/persistence choice, and API-contract ownership. Separate ADRs
would duplicate a tightly coupled Feature 001 decision and are not justified.

## Project Structure

### Documentation (this feature)

```text
specs/001-invoice-capture-processing/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── contracts/
    └── submissions-api.md

docs/adr/
└── 0001-feature-001-hexagonal-processing-boundaries.md
```

### Source Code (repository root)

```text
backend/
├── src/trade_document_automation/
│   ├── domain/
│   │   ├── invoice.py
│   │   ├── validation.py
│   │   └── outcomes.py
│   ├── application/
│   │   └── submissions.py
│   ├── ports/
│   │   ├── extractor.py
│   │   ├── repository.py
│   │   ├── document_storage.py
│   │   └── events.py
│   ├── adapters/
│   │   ├── extraction/local_fixture.py
│   │   ├── persistence/in_memory.py
│   │   ├── storage/in_memory.py
│   │   └── observability/structured_logging.py
│   └── api/
│       ├── app.py
│       ├── routes/submissions.py
│       ├── schemas.py
│       └── dependencies.py
└── tests/
    ├── unit/domain/
    ├── application/
    ├── contract/adapters/
    └── integration/api/

frontend/
├── src/
│   ├── api/submissions.ts
│   ├── features/invoice-capture/
│   ├── components/
│   └── styles/
└── tests/
    ├── components/
    └── e2e/
```

**Structure Decision**: Use the web-application monorepo layout. Backend modules enforce inward
dependencies; the frontend consumes only the documented HTTP API and contract fixtures.

## Constitution Check

### Post-design gate — PASS

The Phase 0–1 artifacts preserve framework- and provider-independent domain logic, define all
required ports and their local adapters, establish the API contract before parallel work, and give
each mandated test layer a validation path. The in-memory adapters are deliberately bounded to the
running process and are not represented as production persistence. No unresolved clarification or
constitution exception remains.

## Complexity Tracking

No constitution violations require justification.
