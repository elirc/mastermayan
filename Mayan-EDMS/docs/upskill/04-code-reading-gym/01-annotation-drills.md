# Annotation Drills

## Drill 1

Excerpt: [`document_type_models.py`](../../../mayan/apps/documents/models/document_type_models.py#L138-L176)

Annotate:

- inputs
- outputs
- invariants
- failure cleanup
- side effects

Self-grade:

- Basic: identifies document and document-file creation.
- Solid: notices rollback on file creation failure.
- Strong: identifies audit/event implications and missing transaction envelope questions.

## Drill 2

Excerpt: [`document_file_models.py`](../../../mayan/apps/documents/models/document_file_models.py#L428-L505)

Annotate derived fields, signals, and expensive operations.

## Drill 3

Excerpt: [`sources/tasks.py`](../../../mayan/apps/sources/tasks.py#L58-L103)

Annotate retry boundary, temp-file lifecycle, and trust assumptions.

## Drill 4

Excerpt: [`file_metadata/tasks.py`](../../../mayan/apps/file_metadata/tasks.py#L31-L48)

Annotate concurrency control and what happens if the lock is unavailable.

## Drill 5

Excerpt: [`ocr/tasks.py`](../../../mayan/apps/ocr/tasks.py#L30-L43)

Annotate fan-out orchestration and predict failure cases.

## Drill 6

Excerpt: [`checkouts/models.py`](../../../mayan/apps/checkouts/models.py#L74-L116)

Annotate business invariant, event semantics, and race questions.

## Drill 7

Excerpt: [`authentication_views.py`](../../../mayan/apps/authentication/views/authentication_views.py#L189-L213)

Annotate session state transitions during MFA escalation.

## Drill 8

Excerpt: [`partial_navigation.js`](../../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L96-L151)

Annotate state variables, request race handling, and user-visible failure modes.
