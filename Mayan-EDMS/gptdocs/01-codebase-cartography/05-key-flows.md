# Key flows

## Flow: upload a new file version

Why it matters: it crosses HTTP, authorization, temporary storage, queueing, database state, and durable file storage.

Open [the API view](../../mayan/apps/documents/api_views/document_file_api_views.py#L28-L65), [serializer contract](../../mayan/apps/documents/serializers/document_file_serializers.py#L71-L140), and [worker](../../mayan/apps/documents/tasks.py#L48-L124).

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | API | view:38-42 | Validate and return 202 | client assumes completion |
| 2 | storage | view:44-47 | Persist shared upload | orphan on enqueue failure |
| 3 | queue | view:49-60 | Send IDs, not file bytes | duplicate delivery |
| 4 | worker | task:64-89 | Load records and call `file_new` | stale/deleted IDs |
| 5 | cleanup | task:90-124 | Delete staging record | cleanup failure |

Authorization occurs before `get_document` returns the parent object at view lines 53-55. Tests show denial as 404 and successful ACL use in [test_document_file_api.py](../../mayan/apps/documents/tests/test_document_file_api.py#L254-L260). Drill: draw the ownership transfer for the staged file. **Strong** includes enqueue failure, retry, and a user-visible job status.

## Flow: download a protected document file

The view scopes the child queryset through the parent document and declares download permission at [document_file_api_views.py](../../mayan/apps/documents/api_views/document_file_api_views.py#L93-L123). The permission adapter delegates object access at [permissions.py](../../mayan/apps/rest_api/permissions.py#L26-L42). The test checks content, filename, MIME type, and audit event at [test_document_file_api.py](../../mayan/apps/documents/tests/test_document_file_api.py#L149-L186).

Interview angle: explain defense in depth—parent scoping prevents arbitrary child lookup, ACL checks authorization, and the event provides auditability. Drill: predict whether changing the child ID to another document’s file can escape the parent queryset; then prove it with a test.

## Flow: ACL-filtered object access

HTTP methods select view-level or object-level permissions in [permissions.py](../../mayan/apps/rest_api/permissions.py#L9-L59). ACL checks construct an authorized queryset and require the object to exist in it at [managers.py](../../mayan/apps/acls/managers.py#L233-L294). Denial appearing as 404 reduces object enumeration, as tested at [test_document_file_api.py](../../mayan/apps/documents/tests/test_document_file_api.py#L100-L129).

Failure modes: forgetting to declare a permission, checking only global roles, or using an unscoped queryset. Drill: review one API view and list view permission, object permission, and queryset scope separately.

## Flow: source action and dynamic input

Enabled sources are selected at [sources/api_views.py](../../mayan/apps/sources/api_views.py#L15-L62); action metadata dynamically adds a file field at [sources/serializers.py](../../mayan/apps/sources/serializers.py#L49-L63). The API combines validated arguments with query parameters before backend dispatch. Investigate whether query values should override validated JSON values; do not call it a bug without a contract test.

Interview angle: this is command dispatch and plugin design. Strong answers discuss allowlists, schema-per-action, and the public contract of backend identifiers.

## Flow: scheduled source ingestion

The task uses a distributed lock, checks enabled state, records errors, and releases the lock in `finally`: [sources/tasks.py](../../mayan/apps/sources/tasks.py#L18-L55). This protects the invariant “one processor per source.” Drill: enumerate crash points and say which are protected by `finally` versus lock expiry.

## Flow: OCR fan-out/fan-in

OCR builds one signature per document page and joins them with a Celery chord at [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L17-L48). Page work retries missing cached images, lock conflicts, and DB operational errors at [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L51-L89). Completion clears error logs and emits an event at [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L92-L125).

Interview angle: discuss parallelism, coordination, partial failure, retry safety, and the latency cost of the slowest page. Drill: design an idempotency test for repeated page tasks.

## Flow: document integrity and deletion

`DocumentFile` stores a SHA-256 checksum and file metadata at [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L51-L129), calculates the digest in blocks at [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L186-L213), and deletes pages, blob, cache partition, then database state at [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L215-L236).

Senior noticing: database and object storage do not share an atomic transaction. A mid-level answer names compensation/reconciliation rather than pretending `transaction.atomic` can roll back S3.
