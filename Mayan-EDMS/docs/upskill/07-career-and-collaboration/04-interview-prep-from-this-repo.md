# Interview Prep from This Repo

Use this repo as proof that you can reason about production Python and practical JavaScript, not just solve toy algorithm questions.

## Python/runtime questions

### Question: Walk me through a file upload in a Django system.

- Junior answer:
  "A view gets a file and saves it."
- Mid-level answer adds:
  ACL-filtered document-type selection in [`web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L42-L50), staged upload creation in [`web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L57-L70), worker rehydration in [`sources/tasks.py`](../../mayan/apps/sources/tasks.py#L64-L87), and rollback in [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L163-L174).
- Senior answer includes:
  request/worker contract, blast radius of synchronous derived work in [`document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py#L460-L484), observability gaps, and a rollback or cleanup story.

### Question: Where would you put business logic in Python?

- Use `Document.file_new()` and `DocumentCheckout.save()` as contrasting examples of domain logic in models.
- Talk about tradeoffs: local discoverability vs side-effect-heavy saves.

### Question: Explain idempotency with a real example.

- Good answer anchor:
  lock-based file metadata processing in [`file_metadata/tasks.py`](../../mayan/apps/file_metadata/tasks.py#L31-L48)

## JavaScript/framework questions

### Question: How do you avoid duplicate event binding in dynamic UIs?

- Junior answer:
  "Bind the click handler once."
- Mid-level answer adds:
  delegated event handling on `body` in [`partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L273-L280) because AJAX replaces content.
- Senior answer includes:
  tradeoffs of delegation, bubbling behavior, and how partial navigation reshapes ownership.

### Question: How do you handle AJAX race conditions?

- Use request cancellation and throttling in [`partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L96-L115).
- Senior addition:
  mention user-visible stale response risks and the distinction between client cancellation and server-side work that may already have started.

## Debugging questions

### Question: An upload returns `202` but nothing shows up. What do you do?

- Strong answer:
  separate request path from worker path, inspect `SharedUploadedFile`, check retry/log behavior in [`sources/tasks.py`](../../mayan/apps/sources/tasks.py#L88-L103), and verify downstream `DocumentFile.save()` side effects.

### Question: A permission bug lets the wrong user mutate data. Where do you look first?

- Strong answer:
  find the first raw object lookup and compare it to established ACL-filter patterns in [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L79-L86).

## System design questions

### Question: How would you design a document-ingestion pipeline?

- Junior:
  "API uploads file and saves it."
- Mid-level:
  mentions async handoff, temp storage, permission gate, background enrichment, retries.
- Senior:
  adds idempotency, observability, archive explosion control, cleanup, and rollback strategy.

### Question: How would you scale OCR?

- Use [`ocr/tasks.py`](../../mayan/apps/ocr/tasks.py#L30-L43) to discuss page-level parallelism.
- Senior addition:
  backpressure, queue isolation, metrics, and completion semantics.

## Code review questions

### Question: What would you flag in a PR that removes ACL-filtered lookup?

- Mid-level:
  calls out authorization regression.
- Senior:
  names it as an IDOR-class risk and suggests preserving the pattern while still reducing duplication.

## "Tell me about a time" prompts

- Tell me about a time you traced a bug across async boundaries.
  Use upload or OCR flows from this repo.
- Tell me about a time you improved a legacy front end safely.
  Use delegated event handling and partial navigation JS examples.
- Tell me about a time you balanced fast delivery with safe permissions.
  Use document upload and cabinet/document access split tests.

## Practice prompts

1. Explain why `user_id` gets passed into tasks instead of a user object.
2. Explain one thing this repo gets right about auth.
3. Critique one design choice kindly and propose a migration path.
