# Architecture critique

Strong choices: modular domain apps; explicit serializers; declarative permission maps; ACL-aware querysets; async heavy work; storage abstraction; audit-event assertions; page-level OCR parallelism.

| Priority | Observation | Status | Improvement / migration |
| --- | --- | --- | --- |
| 1 | Staging row and enqueue are separate | Possible risk: [view](../../mayan/apps/documents/api_views/document_file_api_views.py#L44-L60) | failure test → cleanup guard or durable job/outbox → metrics → staged rollout |
| 2 | Broad exception handling can hide terminal state | Confirmed behavior: [task](../../mayan/apps/documents/tasks.py#L103-L124) | explicit failure state + error taxonomy; preserve cleanup tests |
| 3 | ACL machinery is powerful and complex | Confirmed complexity: [manager](../../mayan/apps/acls/managers.py#L233-L294) | cross-object matrix tests, query-count benchmarks, documented invariants |
| 4 | Toolchain metadata conflicts | Confirmed `setup.py`/`tox.ini` mismatch | decide supported matrix, update generated sources and CI together |
| 5 | Blob/DB deletion is non-atomic | Architectural constraint: [model](../../mayan/apps/documents/models/document_file_models.py#L215-L236) | tombstone state, idempotent cleanup, reconciler, rollback-safe rollout |

Owning this for three months: first baseline failure/queue/ACL metrics; second add regression tests around orphaning, duplicate delivery, and cross-parent access; third make async operation state visible; only then change orchestration. A maintainer should reject a sweeping “service layer” refactor without measured pain, compatibility plan, and incremental tests.

Use this as evidence in [system design](../08-interview-prep/04-system-design-from-this-repo.md).
