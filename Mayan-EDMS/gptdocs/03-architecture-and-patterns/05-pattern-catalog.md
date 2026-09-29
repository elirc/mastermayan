# Pattern catalog

Each card asks you to recognize a shape and its tradeoff, not recite a name.

## Pattern: serializer as boundary contract
Problem: reject malformed HTTP data before domain work. Real example: [document_file_serializers.py](../../mayan/apps/documents/serializers/document_file_serializers.py#L71-L140). It separates create-only upload fields from read-only server fields. Failure: treating serialization as authorization. Drill: classify each field as input, output, or both.

## Pattern: asynchronous acceptance
Problem: keep expensive upload processing out of request latency. Real example: [document_file_api_views.py](../../mayan/apps/documents/api_views/document_file_api_views.py#L38-L60). `202 Accepted` is honest only if status/failure is observable. Drill: sketch a job-status resource.

## Pattern: staging record ownership transfer
Problem: workers cannot safely receive an open request file handle. Real examples: [document_file_api_views.py](../../mayan/apps/documents/api_views/document_file_api_views.py#L44-L60) and [documents/tasks.py](../../mayan/apps/documents/tasks.py#L60-L89). Failure: orphaned staging rows. Drill: define cleanup ownership at each failure point.

## Pattern: ID-only task payload
Problem: keep broker messages small and serializable. Real example: [documents/tasks.py](../../mayan/apps/documents/tasks.py#L52-L72). Failure: IDs become stale before execution. Drill: decide whether “missing” is success, retry, or dead letter.

## Pattern: late model lookup
Problem: avoid import cycles during Django app and worker startup. Real example: [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L22-L28). Failure: typos move errors to runtime. Drill: compare direct imports and app-registry lookup.

## Pattern: bounded retry by exception class
Problem: transient database failures may recover. Real example: [documents/tasks.py](../../mayan/apps/documents/tasks.py#L64-L80). Failure: retrying permanent validation errors or retrying forever. Drill: build a transient/permanent table.

## Pattern: distributed lock around polling
Problem: avoid concurrent processing of one source. Real example: [sources/tasks.py](../../mayan/apps/sources/tasks.py#L18-L55). Failure: expiry shorter than work duration. Drill: explain fencing tokens.

## Pattern: fan-out/fan-in
Problem: parallelize independent page OCR and run completion once. Real example: [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L30-L43). Failure: one poisoned page blocks completion. Drill: propose partial-success semantics.

## Pattern: object-level authorization adapter
Problem: keep view declarations concise while centralizing ACL logic. Real example: [rest_api/permissions.py](../../mayan/apps/rest_api/permissions.py#L9-L59). Failure: missing declarations default to allow. Drill: inspect three views for an explicit permission.

## Pattern: authorized queryset
Problem: authorization must shape collections, not just detail checks. Real example: [acls/managers.py](../../mayan/apps/acls/managers.py#L268-L294). Failure: applying the filter after pagination leaks counts. Drill: identify where filtering must occur.

## Pattern: parent-scoped child lookup
Problem: prevent IDOR across document/file relationships. Real example: [document_file_api_views.py](../../mayan/apps/documents/api_views/document_file_api_views.py#L89-L120). Failure: a later override returns `DocumentFile.objects.all()`. Drill: add a cross-parent test.

## Pattern: audit event after behavior
Problem: retain who did what without coupling API response shape to audit storage. Real evidence: download expects one event at [test_document_file_api.py](../../mayan/apps/documents/tests/test_document_file_api.py#L160-L186). Failure: event loss or double emission on retry. Drill: define exactly-once versus at-least-once expectations.

## Pattern: hook registry
Problem: extend file lifecycle without a giant conditional. Real example: [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L131-L167). Failure: hidden ordering and global mutable registration. Drill: list the hook contract.

## Pattern: storage abstraction
Problem: separate model behavior from filesystem/S3 details. Real example: [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L88-L92). Failure: assuming storage and DB commits are atomic. Drill: design reconciliation.

## Pattern: integrity digest
Problem: detect duplicate or changed bytes without loading the whole file. Real example: [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L186-L213). Failure: confusing integrity with authenticity. Drill: explain why SHA-256 does not prove who uploaded a file.

## Pattern: test mixin DSL
Problem: reuse requests and fixture actions while keeping assertions readable. Real example: [document_file_mixins.py](../../mayan/apps/documents/tests/mixins/document_file_mixins.py#L16-L68). Failure: helpers hide relevant setup. Drill: inline one helper mentally before debugging.
