# Observability and operations

For every major flow answer “how would I know this broke?” Upload: acceptance rate, staging age, enqueue errors, queue age, task duration/failure/retry, orphan count. Download: 4xx/5xx by route, storage latency/errors, authorization denials, audit-event failures. OCR: pages queued/completed/failed, chord age, cache misses, retry exhaustion. Sources: lock contention, last success, documents found, error-log count. ACL: denial rate and query latency without logging sensitive object details.

Structured logs need operation/task ID, object ID, actor ID where policy permits, stage, duration, exception class, and retry count. Metrics show trends; traces connect request→broker→worker; health checks test dependencies shallowly; alerts target user impact. Rollback requires deploy version, migration compatibility, queue payload compatibility, and a runbook.
