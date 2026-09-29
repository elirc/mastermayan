# Side effects, async work, and reliability

Side effects include DB rows, file blobs, cache entries, events, OCR, search updates, and workflow actions. Upload transfers ownership to Celery at [document_file_api_views.py](../../mayan/apps/documents/api_views/document_file_api_views.py#L44-L60). The worker retries operational errors but cleanup behavior differs by exception at [documents/tasks.py](../../mayan/apps/documents/tasks.py#L64-L124).

Idempotency means retrying the same logical command does not duplicate the outcome. A task ID is not automatically an idempotency key. Reliability tools: unique operation keys, transactional outbox for DB-to-broker handoff, bounded exponential retry with jitter, dead-letter visibility, compensation, timeouts, queue backpressure, and reconciliation.

Possible risk: staging creation and enqueue are separate side effects, so enqueue failure may orphan data. Prove with a mocked enqueue failure before proposing an outbox/cleanup change.
