# ADR 0001: Feature 001 processing boundaries and API contract

**Status**: Accepted for Feature 001

## Context

Mobile invoice capture needs reliable approval/review/failure decisions while keeping document
providers, HTTP, and temporary storage replaceable. The frontend and backend may later use separate
worktrees, so their contract needs a single authority first.

## Decision

Use a hexagonal backend: framework-independent domain validation and application use cases depend
on `DocumentExtractor`, `SubmissionRepository`, `DocumentStorage`, and safe-event ports. FastAPI,
the deterministic local extractor, and in-memory adapters are edge adapters. Use the Feature 001
submission REST contract as the shared frontend/backend boundary. Keep accepted image bytes and
safe submission results only in process-local temporary adapters; add durable adapters later if the
product scope requires them.

## Consequences

Azure and other providers can be introduced as extractor adapters without changing business rules.
The first slice supports reconnect/retry only while the process-local temporary state exists; this
is an explicit limitation, not durable production retention. Any shared API-contract change is
serial, coordinated work before parallel implementation proceeds.
