---

description: "Implementation tasks for Mobile Commercial Invoice Capture and Processing"
---

# Tasks: Mobile Commercial Invoice Capture and Processing

**Input**: Design documents from `/specs/001-invoice-capture-processing/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [submissions-api.md](./contracts/submissions-api.md), and
[quickstart.md](./quickstart.md)

**Tests**: Required by the project constitution. Write each listed test before or alongside its
implementation and verify the failure first where practical.

**Organization**: Tasks are grouped by user story so each can be independently delivered and
tested after the shared foundation is complete.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: May run in parallel because it changes different files and has no unfinished dependency.
- **[US#]**: User story served by the task.

## Path Conventions

- Backend code: `backend/src/trade_document_automation/`
- Backend tests: `backend/tests/`
- Frontend code: `frontend/src/`
- Frontend tests: `frontend/tests/`

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish the monorepo skeleton and the stable shared delivery boundary.

- [ ] T001 Create the backend package and test directory skeleton in `backend/src/trade_document_automation/` and `backend/tests/`
- [ ] T002 Configure the root Python project, test paths, Ruff, and mypy in `pyproject.toml`
- [ ] T003 [P] Initialize the minimal React, TypeScript, and Vite frontend configuration in `frontend/package.json`, `frontend/tsconfig.json`, and `frontend/vite.config.ts`
- [ ] T004 [P] Add safe local configuration examples and ignore rules in `.env.example` and `.gitignore`
- [ ] T005 Review and lock the shared submission contract in `specs/001-invoice-capture-processing/contracts/submissions-api.md`
- [ ] T006 Add architecture-boundary guidance for contributors in `docs/adr/0001-feature-001-hexagonal-processing-boundaries.md`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Build framework-independent primitives and composition infrastructure required by all
user stories.

**⚠️ CRITICAL**: Complete this phase and keep the contract stable before creating frontend/backend
worktrees or beginning user-story implementation.

- [ ] T007 Create outcome, field-reliability, finding, submission-ID, and invoice value objects in `backend/src/trade_document_automation/domain/outcomes.py` and `backend/src/trade_document_automation/domain/invoice.py`
- [ ] T008 Create the document-extractor, submission-repository, document-storage, and safe-event ports in `backend/src/trade_document_automation/ports/extractor.py`, `backend/src/trade_document_automation/ports/repository.py`, `backend/src/trade_document_automation/ports/document_storage.py`, and `backend/src/trade_document_automation/ports/events.py`
- [ ] T009 Implement process-local submission repository and temporary document-storage adapters in `backend/src/trade_document_automation/adapters/persistence/in_memory.py` and `backend/src/trade_document_automation/adapters/storage/in_memory.py`
- [ ] T010 [P] Implement metadata-only structured event logging in `backend/src/trade_document_automation/adapters/observability/structured_logging.py`
- [ ] T011 Create FastAPI application composition, dependency wiring, and safe exception-to-error mapping in `backend/src/trade_document_automation/api/app.py` and `backend/src/trade_document_automation/api/dependencies.py`
- [ ] T012 Create shared Pydantic HTTP schemas matching the contract in `backend/src/trade_document_automation/api/schemas.py`
- [ ] T013 [P] Add synthetic image fixture bytes and extractor-result fixtures in `backend/tests/fixtures/images/` and `backend/tests/fixtures/extraction_results.py`
- [ ] T014 Add shared backend test factories and FastAPI test-client setup in `backend/tests/conftest.py` and `backend/tests/factories.py`

**Checkpoint**: Ports, safe adapters, API composition, and synthetic fixtures are ready. The API
contract is the synchronization point after which implementation can be split by ownership.

---

## Phase 3: User Story 1 — Capture, Submit, and Receive an Approval (Priority: P1) 🎯 MVP

**Goal**: A mobile user submits one or more valid invoice images and receives an `approved` result
when trusted required information and totals pass validation.

**Independent Test**: Upload a synthetic valid image through `POST /v1/submissions`, retrieve the
returned reference, and verify `completed` with `approved`, a safe extracted summary, and no
findings; complete the same path in a narrow mobile viewport.

### Tests for User Story 1

- [ ] T015 [P] [US1] Write approval-rule unit tests for required trusted fields and consistent line totals in `backend/tests/unit/domain/test_validation.py`
- [ ] T016 [P] [US1] Write DocumentExtractor contract tests for deterministic trusted synthetic extraction in `backend/tests/contract/adapters/test_local_fixture_extractor.py`
- [ ] T017 [P] [US1] Write submit-and-retrieve use-case tests for an approved submission in `backend/tests/application/test_submissions.py`
- [ ] T018 [P] [US1] Write FastAPI integration tests for accepted upload, `202` response, and approved retrieval in `backend/tests/integration/api/test_submissions.py`
- [ ] T019 [P] [US1] Write mobile capture/result component tests against contract fixtures in `frontend/tests/components/invoice-capture.test.tsx`

### Implementation for User Story 1

- [ ] T020 [US1] Implement framework-independent approval validation and safe summary generation in `backend/src/trade_document_automation/domain/validation.py`
- [ ] T021 [US1] Implement the deterministic synthetic `DocumentExtractor` adapter using fixture-content digests in `backend/src/trade_document_automation/adapters/extraction/local_fixture.py`
- [ ] T022 [US1] Implement submit, process, and retrieve use cases with one final outcome per accepted submission in `backend/src/trade_document_automation/application/submissions.py`
- [ ] T023 [US1] Implement `POST /v1/submissions` and `GET /v1/submissions/{submission_id}` routes exactly as documented in `backend/src/trade_document_automation/api/routes/submissions.py`
- [ ] T024 [US1] Implement the typed frontend submission API client and polling/reconnect retrieval in `frontend/src/api/submissions.ts`
- [ ] T025 [US1] Implement mobile capture/select, processing-state, and approved-result UI in `frontend/src/features/invoice-capture/InvoiceCapturePage.tsx` and `frontend/src/styles/invoice-capture.css`
- [ ] T026 [US1] Wire the mobile capture page into the frontend application entry point in `frontend/src/App.tsx` and `frontend/src/main.tsx`

**Checkpoint**: The MVP upload → processing → approved-result journey is independently functional
through the documented API and mobile UI.

---

## Phase 4: User Story 2 — Understand a Required Human Review (Priority: P1)

**Goal**: A user and reviewer can understand a `review_required` result when extraction is
sufficient but applicable business information is missing, ambiguous, low-confidence, or
inconsistent.

**Independent Test**: Submit synthetic fixtures for a missing required field and an inconsistent
total, then verify `review_required` plus safe field/rule, reason, and condition in the response
and mobile result.

### Tests for User Story 2

- [ ] T027 [P] [US2] Write domain unit tests for missing, ambiguous, low-confidence, and inconsistent review findings in `backend/tests/unit/domain/test_validation.py`
- [ ] T028 [P] [US2] Write application tests that persist safe review context and never approve an untrusted required field in `backend/tests/application/test_submissions.py`
- [ ] T029 [P] [US2] Write FastAPI integration tests for `review_required` response fields in `backend/tests/integration/api/test_submissions.py`
- [ ] T030 [P] [US2] Write review-result mobile component tests using documented safe findings in `frontend/tests/components/invoice-capture.test.tsx`

### Implementation for User Story 2

- [ ] T031 [US2] Extend domain validation to create review findings and enforce optional shipment/reference semantics in `backend/src/trade_document_automation/domain/validation.py`
- [ ] T032 [US2] Extend the local fixture extraction results for reviewable conditions in `backend/src/trade_document_automation/adapters/extraction/local_fixture.py`
- [ ] T033 [US2] Map review findings and next actions to the safe HTTP response in `backend/src/trade_document_automation/api/schemas.py` and `backend/src/trade_document_automation/api/routes/submissions.py`
- [ ] T034 [US2] Implement the `review_required` explanation and review-context display in `frontend/src/features/invoice-capture/InvoiceResult.tsx`

**Checkpoint**: Review-required decisions are distinct from approval and expose only safe,
actionable review context.

---

## Phase 5: User Story 3 — Correct an Invalid Upload (Priority: P2)

**Goal**: A user gets clear correction feedback for invalid files before extraction, with no
processing outcome assigned.

**Independent Test**: Submit empty, oversized, unsupported, and MIME/extension/signature-mismatch
synthetic files and verify the documented 400/413/415/422 safe errors, no extractor invocation,
and a replacement-file UI path.

### Tests for User Story 3

- [ ] T035 [P] [US3] Write upload-validation unit tests for extension, declared MIME, signature, emptiness, and 10 MiB limits in `backend/tests/unit/domain/test_upload_validation.py`
- [ ] T036 [P] [US3] Write use-case tests proving rejected uploads create no submission or processing outcome in `backend/tests/application/test_submissions.py`
- [ ] T037 [P] [US3] Write FastAPI integration tests for all documented upload-validation status codes in `backend/tests/integration/api/test_upload_validation.py`
- [ ] T038 [P] [US3] Write mobile invalid-upload feedback and replacement-selection tests in `frontend/tests/components/invoice-capture.test.tsx`

### Implementation for User Story 3

- [ ] T039 [US3] Implement multi-signal image upload validation and safe rejection types in `backend/src/trade_document_automation/domain/upload_validation.py`
- [ ] T040 [US3] Integrate streaming size enforcement and upload-error mapping before storage/extraction in `backend/src/trade_document_automation/api/routes/submissions.py` and `backend/src/trade_document_automation/api/schemas.py`
- [ ] T041 [US3] Implement actionable invalid-upload feedback and replacement selection in `frontend/src/features/invoice-capture/InvoiceCapturePage.tsx`

**Checkpoint**: Invalid files are rejected safely before processing and cannot be confused with a
processing failure.

---

## Phase 6: User Story 4 — Distinguish a Processing Failure (Priority: P2)

**Goal**: A user can distinguish technical `processing_failed` from `review_required`, retry the
same temporary submission, or replace its images.

**Independent Test**: Make the deterministic extractor return a technical failure; verify a final
`processing_failed` response with a safe reason, successful `POST .../retry` behavior, and distinct
mobile retry/replacement actions.

### Tests for User Story 4

- [ ] T042 [P] [US4] Write use-case tests that map extractor/storage technical failures to `processing_failed` and permit only failed-submission retries in `backend/tests/application/test_submissions.py`
- [ ] T043 [P] [US4] Write extractor-adapter contract tests for deterministic technical failures in `backend/tests/contract/adapters/test_local_fixture_extractor.py`
- [ ] T044 [P] [US4] Write FastAPI integration tests for failed retrieval and retry conflict behavior in `backend/tests/integration/api/test_submission_retry.py`
- [ ] T045 [P] [US4] Write mobile processing-failure, retry, and replacement-action tests in `frontend/tests/components/invoice-capture.test.tsx`

### Implementation for User Story 4

- [ ] T046 [US4] Implement technical-failure completion, attempt incrementing, and retry eligibility in `backend/src/trade_document_automation/application/submissions.py` and `backend/src/trade_document_automation/adapters/persistence/in_memory.py`
- [ ] T047 [US4] Implement the documented retry endpoint and safe conflict/not-found errors in `backend/src/trade_document_automation/api/routes/submissions.py`
- [ ] T048 [US4] Add deterministic extraction-failure fixtures in `backend/src/trade_document_automation/adapters/extraction/local_fixture.py` and `backend/tests/fixtures/extraction_results.py`
- [ ] T049 [US4] Implement distinct processing-failure feedback, retry, and replace-images actions in `frontend/src/features/invoice-capture/InvoiceResult.tsx` and `frontend/src/api/submissions.ts`

**Checkpoint**: Technical failure is never presented as review-required; retry behavior preserves
the submission reference and replacement starts a new submission.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Complete production-oriented checks that span all user stories.

- [ ] T050 [P] Add metadata-only processing, validation, and outcome observability assertions in `backend/tests/integration/api/test_observability.py`
- [ ] T051 [P] Add the mobile Playwright capture-to-result flows in `frontend/tests/e2e/invoice-capture.spec.ts`
- [ ] T052 Add accessibility labels, focus behavior, and small-screen layout checks in `frontend/src/features/invoice-capture/InvoiceCapturePage.tsx`, `frontend/src/features/invoice-capture/InvoiceResult.tsx`, and `frontend/src/styles/invoice-capture.css`
- [ ] T053 Verify all API examples and quickstart commands against the implementation in `specs/001-invoice-capture-processing/contracts/submissions-api.md` and `specs/001-invoice-capture-processing/quickstart.md`
- [ ] T054 Run the full backend and frontend quality gates documented in `specs/001-invoice-capture-processing/quickstart.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 — Setup**: Starts immediately. T005 is the required contract-stabilization gate.
- **Phase 2 — Foundational**: Depends on Phase 1; blocks all user stories and worktree splitting.
- **Phase 3 — US1 (P1)**: Depends on Phase 2; delivers the MVP.
- **Phase 4 — US2 (P1)**: Depends on Phase 2 and extends the shared US1 processing path.
- **Phase 5 — US3 (P2)**: Depends on Phase 2; its validation route changes must coordinate with
  US1 route ownership if work occurs concurrently.
- **Phase 6 — US4 (P2)**: Depends on Phase 2 and the shared submission processing path from US1.
- **Phase 7 — Polish**: Depends on the stories selected for delivery.

### User Story Dependency Graph

```text
Setup → Foundational → US1 (MVP) → US2
                    ├────────────→ US3
                    └────────────→ US4
US1 + US2 + US3 + US4 → Polish
```

### Parallel Opportunities

- During Setup, T003 and T004 can proceed after T001 without editing the contract or backend core.
- In each story, its listed `[P]` test tasks can be written in parallel because they target
  different test scopes; reconcile shared test files serially before merging.
- After T005 and Phase 2 are complete, one owner can take backend tasks, one can take frontend
  tasks, and one independent reviewer can inspect tests/security without editing those owners'
  files. Shared contract, architecture, and route files remain single-owner work.
- US3's frontend feedback can proceed in parallel with backend validation only after the documented
  error fixtures are agreed; US4's frontend result UI can proceed in parallel with backend retry
  logic under the same constraint.

## Parallel Example: User Story 1

```text
# After Phase 2, before implementation changes:
T015 approval-rule unit tests
T016 DocumentExtractor adapter contract tests
T017 submit/retrieve use-case tests
T018 FastAPI integration tests
T019 mobile component tests

# After shared API fixtures are settled:
T021 local extractor adapter (backend owner)
T024 typed frontend client (frontend owner)
```

## Implementation Strategy

### MVP First

1. Complete Phases 1 and 2 with the contract reviewed and frozen.
2. Complete Phase 3 and run its independent backend and mobile tests.
3. Demonstrate only the valid-upload → processing → approved path.

### Incremental Delivery

1. Add Phase 4 to make uncertain documents safely reviewable.
2. Add Phase 5 to protect the system and user from invalid uploads.
3. Add Phase 6 to distinguish technical failure and supply recovery paths.
4. Complete Phase 7, including Playwright mobile flows and the quickstart quality gates.

### Format Validation

All 54 implementation tasks use `- [ ]`, a sequential task ID, an optional `[P]` marker only for
parallel work, a user-story label on every story task, and one or more exact repository file paths.
