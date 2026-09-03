# Feature 001 Validation Quickstart

This guide validates the planned vertical slice using only synthetic image fixtures. See the
[data model](./data-model.md) and [API contract](./contracts/submissions-api.md) for expected
states and payloads.

## Prerequisites

- Python 3.12 and `uv`.
- Node.js LTS and npm once the proposed frontend is introduced.
- No cloud credentials, provider SDKs, real invoices, or secrets.

## Backend verification

From the repository root, synchronize the Python environment and run the backend checks after
implementation:

```bash
uv sync --all-groups
uv run pytest backend/tests/unit/domain backend/tests/application backend/tests/contract/adapters backend/tests/integration/api
uv run ruff check backend
uv run mypy backend/src
```

Start the local service using the implementation's documented application entry point, then submit
synthetic fixtures from the API integration tests. Validate these observable cases:

1. A trusted fixture with the required fields and consistent totals returns `202`, then `approved`.
2. Missing, ambiguous, low-confidence, or inconsistent required data returns `review_required`
   with safe field/rule findings.
3. A fixture that makes the local extractor fail returns `processing_failed` and supports retry.
4. An empty, mismatched, unsupported, or >10 MiB file receives the documented upload error and no
   processing outcome.
5. Fetching the submission reference after a simulated reconnect returns its current or final
   result without creating another submission.

## Frontend verification

After the frontend exists, run its unit/component tests and then execute the critical mobile flow
in a narrow viewport:

```bash
npm --prefix frontend test
npm --prefix frontend exec playwright test
```

Verify capture/select → submit → visible processing state → final outcome. Exercise invalid upload,
review-required feedback, processing-failed retry/replacement, and reconnect result retrieval.
Playwright is introduced with the UI; it is not a prerequisite for planning.

## Quality gates

Before a milestone commit, run the relevant test, lint, and type-check commands; inspect HTTP
responses against the contract; and confirm logs contain only opaque references, outcome/finding
codes, and timing metadata. Do not commit synthetic fixtures containing real commercial data.
