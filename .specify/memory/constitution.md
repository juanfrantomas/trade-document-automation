<!--
Sync Impact Report
- Version change: unversioned template → 1.0.0
- Modified principles: none (initial constitution)
- Added sections: Processing Integrity and User Experience; Delivery Collaboration and Quality;
  Governance
- Removed sections: none
- Follow-up TODOs: none
-->
# trade-document-automation Constitution

## Core Principles

### I. Hexagonal Architecture
Domain models and business rules MUST be independent of frameworks, cloud vendors, databases, UI,
and external services. Backend code MUST separate domain, application/use cases, ports, and
adapters. External systems MUST use explicit ports and adapters; FastAPI MUST remain at the edge.

### II. Vendor Independence
Document extraction MUST be a port. Azure Document Intelligence, Google Document AI, AWS Textract,
Tesseract, local, and mock extractors MUST be interchangeable adapters without changing domain
logic. Persistence, file storage, notifications, workflow triggers, and external AI services MUST
use the same abstraction rule.

### III. Specification-Driven Development
Every important feature MUST follow GitHub Spec Kit: specification, technical plan, tasks, then
implementation. Its `spec.md`, `plan.md`, and `tasks.md` MUST exist before implementation.
Specifications state WHAT and WHY; technical details belong in plans. Material behavior changes
MUST update the relevant specification in the same milestone.

### IV. Practical Test-Driven Development
Domain rules MUST have unit tests; application use cases MUST have use-case tests; and external
adapters MUST have contract tests where applicable. FastAPI endpoints MUST have integration tests;
mobile UI flows MUST have Playwright tests when introduced. Tests MUST be written before or
alongside implementation, assert observable behavior, and cover every critical business rule.
Manual testing alone is never sufficient for a critical business rule.

### V. Secure by Design
Secrets MUST never be committed; runtime secrets MUST use environment variables, and `.env.example`
MAY contain only non-real values. Uploads MUST validate allowed MIME type, extension, and size.
Real invoices, FedEx documents, customer data, and confidential documents MUST NOT enter this
repository; demos and tests MUST use synthetic fixtures. Security-sensitive changes require
explicit review. Cloud storage, AI providers, authentication, and public upload endpoints MUST
include threat modeling.

## Processing Integrity and User Experience

### VI. Human Oversight and Processing Outcomes
Every processing attempt MUST end as `approved`, `review_required`, or `processing_failed`.
Ambiguous, incomplete, low-confidence, or business-rule-invalid documents MUST NOT silently be
approved. The design MUST support human review and retain automated decisions and relevant
validation failures.

### VII. Traceability and Observability
Important processing steps, validation results, and decisions MUST be traceable without exposing
sensitive document content or secrets. Errors MUST be explicit and diagnosable. Logging MUST use
safe identifiers and metadata so operators can explain approval, review, or failure outcomes.

### VIII. Mobile-First Web Experience
Mobile capture or upload of a commercial invoice is the primary flow. The frontend MUST prioritize
mobile usability, clear processing state, actionable validation feedback, and review status.
Desktop enhancements MUST NOT compromise this flow.

## Delivery Collaboration and Quality

### IX. API Contract Before Parallel Delivery
Before frontend and backend work is split, their API contract MUST be documented, covering request
and response models, status values, validation errors, and expected HTTP behavior. Breaking
contract changes require coordination between the frontend and backend owners.

### X. Multi-Agent and Git Worktree Collaboration
Parallel work is allowed only for genuinely independent tasks. Each feature or worktree MUST have
exactly one owner, and agents MUST NOT edit the same files concurrently. Frontend and backend MAY
use separate worktrees only after the shared API contract is stable. One agent MAY implement while
another independently reviews tests, architecture, security, or specification consistency, provided
the reviewer does not modify the same files concurrently. Shared architecture, governance, API
contract, and other shared files MUST be changed serially. Every worktree MUST use a dedicated Git
branch. Destructive Git operations are prohibited.

### XI. Git and Quality Gates
Commits MUST be complete, verified logical milestones and use `type(scope): short summary`.
Before a commit, contributors MUST run relevant tests, linting, type checking, and status checks.
Partially working milestones MUST NOT be committed, and destructive Git commands MUST be avoided.

### XII. Portfolio Quality and Initial Constraints
The project MUST demonstrate production-oriented engineering, preferring simple, explicit,
testable designs over needless complexity. Long-lived architectural decisions MUST be recorded
concisely in `docs/`; repository and documentation MUST support technical interview discussion.
The initial stack is Python 3.12 with `uv`, FastAPI, and a mobile web frontend in a monorepo with
`backend/`, `frontend/`, `specs/`, and `docs/`. Azure integrations are out of scope initially;
local or mock document extraction and automated tests MUST come first.

## Governance

This constitution supersedes conflicting practices. Specs, plans, tasks, reviews, and commits MUST
verify compliance. Exceptions require an explicit, time-bounded rationale in a plan or decision
record plus security and architectural review.

Amendments MUST be documented here and reviewed for affected specs, plans, tasks, contracts,
tests, and documentation, with needed migration work. Semantic versioning applies: MAJOR for
backward-incompatible principle removal or redefinition, MINOR for added or materially expanded
governance, and PATCH for non-semantic clarification. Compliance MUST be reviewed during feature
planning, implementation review, and before milestone commits.

**Version**: 1.0.0 | **Ratified**: 2026-09-03 | **Last Amended**: 2026-09-03
