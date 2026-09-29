# Mid-Level Feature Tickets

## Ticket 1: Add structured upload trace logging
Design notes required first. Risk: low-to-medium. Rollback: remove extra log fields.

## Ticket 2: Add an orphan `SharedUploadedFile` sweeper command
Design notes required first. Risk: deleting active temp uploads. Rollback: dry-run mode or feature flag.

## Ticket 3: Add an operator-facing OCR backlog health command
Design notes required first. Risk: misleading health signal if too shallow. Rollback: keep internal only.

## Ticket 4: Add stronger archive-upload guardrails
Design notes required first. Risk: breaking valid archive workflows. Rollback: setting-gated limits.

## Ticket 5: Add API contract tests for document type change flow
Design notes required first. Risk: low. Rollback: test-only.

## Ticket 6: Surface parsing/OCR error logs in a clearer admin or UI location
Design notes required first. Risk: exposing internal details too broadly. Rollback: staff-only or admin-only view.

## Ticket 7: Add targeted metrics around index rebuild task volume
Design notes required first. Risk: noisy instrumentation. Rollback: disable emission.

## Ticket 8: Improve checkout conflict messaging with actor and expiration context
Design notes required first. Risk: information disclosure. Rollback: revert to generic text.

## Ticket 9: Add a developer docs page for app-local signal chains
Design notes required first. Risk: docs drift. Rollback: keep scope small and representative.

## Ticket 10: Add a batch-safe permission helper for repeated ACL-filter patterns
Design notes required first. Risk: over-abstraction around security-critical code. Rollback: keep helper internal and migrate incrementally.
