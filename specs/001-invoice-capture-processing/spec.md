# Feature Specification: Mobile Commercial Invoice Capture and Processing

**Feature Branch**: `001-invoice-capture-processing`

**Created**: 2026-09-03

**Status**: Draft

**Input**: User description: "Mobile Commercial Invoice Capture and Processing"

## Actors

- **Mobile user**: Captures or selects an invoice image, submits it, and reads the outcome.
- **Reviewer**: Uses the supplied review context to understand why a document needs human review.
  The reviewer decision interface is outside this feature's scope.

## Clarifications

### Session 2026-09-03

- Q: Which image formats and maximum file size should the mobile upload accept? → A: JPEG, PNG, or HEIC, maximum 10 MiB.
- Q: Which rules must an invoice pass to receive `approved` in the initial release? → A: Require invoice number, date, seller, buyer, currency, total, and consistent line-item totals.
- Q: If a user loses connectivity or leaves the page after submitting an invoice, how should they recover the result? → A: Resume and show the submitted document's final result after reconnection.
- Q: After a `processing_failed` result, which retry action should the mobile user have? → A: Retry the same image or select a replacement image.
- Q: How many images may a user include in one invoice submission? → A: Any number of images per submission.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Capture, Submit, and Receive an Approval (Priority: P1)

A mobile user captures a photo of a commercial invoice or selects an existing image, confirms that
it is acceptable, submits it, and receives an approved result when the information can be
reliably extracted and passes all applicable business validation rules.

**Why this priority**: This is the core value: a user can turn a mobile invoice image into a
trusted processing decision with minimal effort.

**Independent Test**: Submit an acceptable synthetic invoice image with reliable, valid required
information and verify that the user sees `approved` with a clear result.

**Acceptance Scenarios**:

1. **Given** a mobile user can access the capture flow, **When** they capture or select an
   acceptable invoice image and submit it, **Then** the system shows a clear in-progress status.
2. **Given** an accepted document has reliably extracted information that passes every applicable
   business rule, **When** processing completes, **Then** the system reports exactly `approved`.
3. **Given** an approved result, **When** the user views it on a small screen, **Then** the outcome
   and its summary are readable without requiring a desktop-only interaction.

---

### User Story 2 - Understand a Required Human Review (Priority: P1)

A mobile user receives a clear `review_required` result when processing completes but applicable
business information is missing, ambiguous, inconsistent, or insufficiently reliable. A reviewer
can understand the reason without needing to inspect internal system details.

**Why this priority**: The feature must prevent untrustworthy documents from being silently
approved while preserving a path for human judgment.

**Independent Test**: Submit a synthetic invoice with an applicable required field that is missing,
ambiguous, inconsistent, or unreliable; verify the `review_required` result and review context.

**Acceptance Scenarios**:

1. **Given** extraction completes but an applicable required field cannot be trusted, **When** the
   business validation is evaluated, **Then** the system reports exactly `review_required`, not
   `approved` or `processing_failed`.
2. **Given** a document requires review, **When** a reviewer reads its review context, **Then** it
   identifies the affected field or rule, the reason for review, and the relevant reliability or
   consistency condition without exposing internal implementation details.
3. **Given** shipment or reference information is absent but not applicable to the document,
   **When** validation is evaluated, **Then** its absence alone does not require review.

---

### User Story 3 - Correct an Invalid Upload (Priority: P2)

A mobile user receives understandable feedback before processing when the selected file is not an
acceptable invoice image, and can choose another file without entering the processing workflow.

**Why this priority**: Early feedback protects the processing flow and avoids presenting a file
acceptance problem as a document-processing decision.

**Independent Test**: Select a file that fails an acceptance constraint and verify safe feedback,
no processing attempt, and the ability to select a replacement.

**Acceptance Scenarios**:

1. **Given** a user selects a file with an unsupported type, extension, or size, **When** the
   selection is validated, **Then** the system explains that the file cannot be accepted and does
   not start processing.
2. **Given** a file fails acceptance validation, **When** the user reads the feedback, **Then** it
   states the correction needed without disclosing internal system details.

---

### User Story 4 - Distinguish a Processing Failure (Priority: P2)

A mobile user can distinguish an inability to complete processing from a document that completed
processing but needs human review.

**Why this priority**: Users need an honest outcome and safe next step when the workflow cannot be
reliably completed.

**Independent Test**: Submit an acceptable synthetic image for which extraction or processing
cannot be reliably completed and verify the `processing_failed` result and safe feedback.

**Acceptance Scenarios**:

1. **Given** a document passed file acceptance but processing cannot reliably complete, **When**
   the workflow ends, **Then** the system reports exactly `processing_failed`.
2. **Given** a `processing_failed` result, **When** the user views it, **Then** the feedback is
   distinguishable from `review_required`, avoids internal details, and lets them retry the same
   accepted submission or select replacement images.

---

### Edge Cases

- The mobile device has no usable camera, access is declined, or capture is cancelled: the user can
  select an existing image and receives clear feedback if no file is selected.
- A selected file has an allowed-looking extension but unsupported content or a mismatched file
  type: it fails acceptance validation before processing.
- The image is accepted but is unreadable, incomplete, or contains no reliably extractable invoice
  information: it results in `processing_failed`, not `review_required`.
- An optional field, such as shipment or reference information, is absent: it is not treated as a
  failure unless it is applicable and required by the relevant business rule.
- Required extracted values conflict, such as totals that are inconsistent with available line
  items: processing completes as `review_required` with the conflict identified.
- The user leaves or loses connectivity after submission: after reconnecting, they can resume the
  same submission and see its completed final result; no visible result may imply approval unless
  a completed `approved` outcome is available.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST allow a mobile user to capture commercial invoice images with the
  device camera or select one or more existing images from the device for one submission.
- **FR-002**: The capture-and-submit flow MUST be usable on a mobile browser, with outcome,
  processing state, feedback, and next actions readable on a small screen.
- **FR-003**: Before processing, the system MUST accept only non-empty JPEG, PNG, or HEIC images
  no larger than 10 MiB each, and MUST validate the allowed extension and matching file type of
  every image in a submission.
- **FR-004**: A file that fails acceptance validation MUST receive understandable, safe feedback,
  MUST NOT enter processing, and MUST NOT be classified as `processing_failed`.
- **FR-005**: The system MUST allow a user to submit an accepted document and MUST present a clear
  in-progress status until a final result is available.
- **FR-006**: For an accepted document, the system MUST attempt to obtain applicable structured
  commercial invoice information, including invoice number, invoice date, seller/exporter,
  buyer/importer, currency, totals, line items, shipment or reference information when present,
  and other relevant fields.
- **FR-007**: The system MUST represent information that is absent, ambiguous, inconsistent, or
  insufficiently reliable in a way that can be evaluated by business validation rules.
- **FR-008**: The system MUST evaluate extracted information against the applicable business
  validation rules and MUST produce exactly one final processing outcome for each accepted,
  submitted document: `approved`, `review_required`, or `processing_failed`.
- **FR-009**: The system MUST report `approved` only when processing completes reliably and every
  applicable required business rule passes with trusted information.
- **FR-010**: The system MUST report `review_required` when processing completes but applicable
  required information is missing, ambiguous, inconsistent, or insufficiently reliable, or when an
  applicable business rule cannot be satisfied with trusted information.
- **FR-011**: The system MUST report `processing_failed` only when it cannot reliably complete the
  processing workflow; it MUST NOT use `processing_failed` for a business validation issue that is
  eligible for human review.
- **FR-012**: The user-facing result MUST clearly state the final outcome, distinguish it from file
  acceptance feedback, and provide a safe, understandable next action when action is possible.
- **FR-013**: For `review_required`, the system MUST retain and present reviewer context containing
  the document submission reference, the affected field or rule, the reason for review, and the
  relevant missing, ambiguous, inconsistent, or reliability condition.
- **FR-014**: For every final outcome, the system MUST retain enough safe processing and validation
  context to explain why the result was reached without requiring retention of sensitive raw
  document content solely for traceability.
- **FR-015**: User-facing errors and review context MUST NOT expose secrets, internal processing
  details, or more document content than is needed to understand the result.
- **FR-016**: Development and testing for this feature MUST use synthetic documents only. Real
  customer, FedEx, or other confidential commercial documents are outside the development and test
  scope.
- **FR-017**: The primary capture-and-submit journey MUST avoid unnecessary steps between selecting
  an acceptable image and receiving the processing result.
- **FR-018**: After a user reconnects or returns to the capture flow, the system MUST resume the
  same submitted document and display its available final result without requiring a duplicate
  submission.
- **FR-019**: For a `processing_failed` outcome, the system MUST let the user retry the accepted
  submission or capture or select replacement images.
- **FR-020**: A document submission MAY contain any number of accepted images; the system MUST
  process them together as one commercial invoice submission and issue one final outcome.

### Validation and Processing Rules

| Classification | When it applies | Required behavior |
| --- | --- | --- |
| Upload validation failure | The selected file fails a file acceptance constraint before processing. | Give safe correction feedback; do not start processing; do not assign a processing outcome. |
| `processing_failed` | An accepted document cannot complete the processing workflow reliably. | Give a distinct failure result and safe next action; do not treat it as a business-rule failure. |
| `review_required` | Processing completes, but applicable business information cannot be trusted or a rule cannot be validated. | Preserve the review reason and context; do not approve the document. |
| `approved` | Processing completes reliably and all applicable required business rules pass with trusted information. | Give a clear approval result and retain safe decision context. |

- A field is required only when the applicable business rules require it for the submitted
  document. The absence of a non-applicable or optional field MUST NOT prevent approval.
- For the initial release, approval requires trusted invoice number, invoice date, seller/exporter,
  buyer/importer, currency, and invoice total; where line items are present, their totals MUST be
  consistent with the invoice total.
- A document with untrusted applicable required business information MUST be routed to
  `review_required`; it MUST never silently become `approved`.
- A business validation issue that a human can assess is a `review_required` condition, not a
  processing failure.

### Key Entities

- **Document submission**: A user's selected or captured invoice images and their collective
  acceptance state.
- **Extracted invoice information**: The applicable structured business information identified from
  a document, including values and their availability or reliability condition.
- **Processing outcome**: The single final classification for an accepted submission: `approved`,
  `review_required`, or `processing_failed`.
- **Validation finding**: A business-rule result or processing condition that supports an outcome
  and, where relevant, explains what needs review.
- **Review context**: The safe information a reviewer needs to understand a review-required result.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In representative mobile usability tests, at least 90% of participants can capture
  or select an acceptable invoice image, submit it, and identify the displayed result without
  assistance.
- **SC-002**: Under normal operating conditions, at least 95% of accepted submissions reach a
  visible final result within 60 seconds of submission.
- **SC-003**: 100% of acceptance-test cases with an invalid upload receive file-acceptance feedback
  without entering processing or receiving a processing outcome.
- **SC-004**: 100% of acceptance-test cases for a trusted, business-rule-valid document result in
  `approved`; 100% of cases with reviewable missing, ambiguous, inconsistent, or unreliable
  applicable information result in `review_required`; and 100% of cases where processing cannot
  reliably complete result in `processing_failed`.
- **SC-005**: 100% of `review_required` acceptance-test cases expose the affected field or rule and
  the reason for review in reviewer context.
- **SC-006**: 100% of development and test fixtures used for this feature are synthetic.

## Assumptions

- The initial feature serves a user who submits one invoice at a time, with any number of images in
  that submission, and receives the result in the current interaction; submission history and bulk
  processing are not required.
- Applicable business rules define which information is required for a particular invoice context;
  not every conceptual invoice field is universally required.
- Image capture and image selection are in scope; support for non-image document formats is not
  required for this feature.
- A reviewer can receive the specified review context through a later or existing review process;
  this feature does not define the detailed reviewer interface or the human decision process.
- Only synthetic documents are available for development and testing.

## Out of Scope

- Detailed reviewer screens, reviewer decisions, and remediation workflows after a review result.
- Authentication, user authorization, document-sharing policies, bulk submission, and submission
  history.
- Support for non-image document formats.
- Long-term retention policy, storage design, provider selection, and implementation of document
  extraction or processing.
- Technical interface design, data-storage design, and frontend implementation choices.
