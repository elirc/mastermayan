# Timed debugging and code-review rounds

## Q33: An upload returns 202 but never appears. Debug it.
Round: debugging. Repo anchors: [view](../../mayan/apps/documents/api_views/document_file_api_views.py#L38-L60), [task](../../mayan/apps/documents/tasks.py#L48-L124). Mid answer splits before/after enqueue and probes staging, queue, worker, storage. Senior adds correlation, terminal state, retry/idempotency. Time: 12 minutes.

## Q34: OCR is stuck for one 400-page file only.
Round: debugging. Repo anchor: [OCR tasks](../../mayan/apps/ocr/tasks.py#L17-L89). Mid checks page/cache retry and chord backend. Senior checks poison page, tail latency, fairness, partial status. Time: 10 minutes.

## Q35: A user with a role receives 404 downloading one file.
Round: debugging. Repo anchors: [permission adapter](../../mayan/apps/rest_api/permissions.py#L26-L42), [ACL manager](../../mayan/apps/acls/managers.py#L233-L294). Mid verifies parent membership, permission kind, inheritance, trashed state. Senior uses query evidence and preserves non-enumeration. Time: 10 minutes.

## Q36: Storage usage rises after document deletions.
Round: debugging. Repo anchor: [delete sequence](../../mayan/apps/documents/models/document_file_models.py#L215-L236). Mid injects failures per boundary. Senior designs reconciliation metrics and safe repair. Time: 12 minutes.

## Q37: Review a PR replacing the parent-scoped file queryset with global lookup.
Round: code review. Repo anchor: [current querysets](../../mayan/apps/documents/api_views/document_file_api_views.py#L89-L120). Blocking: IDOR/canonical relationship. Mid requests wrong-parent test. Senior audits list/bulk/cache siblings. Time: 8 minutes.

## Q38: Review a PR that retries all upload exceptions forever.
Round: code review. Repo anchor: [current exception branches](../../mayan/apps/documents/tasks.py#L90-L124). Blocking: poison loop/duplicate effects. Ask for classification, max attempts, terminal state, metrics. Time: 8 minutes.

## Q39: Review a PR hashing the entire file in memory.
Round: code review. Repo anchor: [block loop](../../mayan/apps/documents/models/document_file_models.py#L186-L213). Important/blocking depends size limits. Ask for representative memory benchmark and preserve digest behavior. Time: 6 minutes.

## Q40: Review a PR caching document lists globally for five minutes.
Round: code review. Repo anchor: [identity-specific ACL filtering](../../mayan/apps/acls/managers.py#L268-L294). Blocking: data leak. Senior discusses permission change invalidation and shared SSR/proxy caches. Time: 8 minutes.

Rubric per round: 0–1 random guesses; 2 reproduces and splits a boundary; 3 uses an evidence-based hypothesis and regression test; 4 adds invariant, security/reliability, observability, and lowest-risk fix.
