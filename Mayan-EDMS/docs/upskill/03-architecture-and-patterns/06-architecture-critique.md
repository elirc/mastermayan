# Architecture Critique

## Strongest design choices

- Object-level authorization is a first-class concern, not a scattered afterthought.
- Async document-processing stages are explicit and app-local.
- Docker, staging, and CI reflect real operational concerns instead of only unit tests.

## Confirmed tradeoffs

- Model-centric orchestration keeps logic discoverable near data but increases save-time side effects.
- Signals and callbacks reduce duplication but hide causal chains.
- Generic ACLs are flexible but harder to reason about than type-specific permission tables.

## Risks and hypotheses

### Confirmed risk

- `DocumentFile.save()` does expensive work synchronously after insert [`document_file_models.py`](../../../mayan/apps/documents/models/document_file_models.py#L460-L484).
  Likely impact: latency and harder debugging around upload failures.

### Hypothesis to investigate

- Signal-driven indexing may amplify load during bulk document operations [`document_indexing/handlers.py`](../../../mayan/apps/document_indexing/handlers.py#L71-L83).
  Confidence: medium.

### Hypothesis to investigate

- Shared upload cleanup may be vulnerable to orphan accumulation when downstream failures happen after staging.
  Evidence: temp upload deletion occurs after `handle_file_object_upload()` returns in [`sources/tasks.py`](../../../mayan/apps/sources/tasks.py#L95-L103).

## If I owned this repo for 3 months

1. Add stronger observability around upload, parsing, OCR, and indexing queues.
2. Reduce hidden coupling by documenting or consolidating key signal chains.
3. Evaluate whether some `DocumentFile.save()` work should move behind async post-commit tasks.

## Migration-minded improvement ideas

| Recommendation | Migration path | Test strategy |
| --- | --- | --- |
| instrument upload/OCR/indexing latencies | add structured logging and metrics first | assert log fields in targeted tests where possible |
| isolate heavy derived-file work | introduce opt-in task path behind setting | compare upload behavior and add regression tests |
| tighten temp-upload cleanup | periodic orphan sweeper or finally-block improvement | failure-injection test around upload task |
