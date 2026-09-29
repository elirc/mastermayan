# Risk register

These are investigation targets, not confirmed bugs.

| Risk | Evidence | Impact | Likelihood | Suggested test/fix | Confidence |
| --- | --- | --- | --- | --- | --- |
| Staging orphan on enqueue failure | [separate create/enqueue](../../mayan/apps/documents/api_views/document_file_api_views.py#L44-L60) | storage growth | Medium | mock failure; cleanup/outbox/reconciler | Medium |
| Duplicate upload on task redelivery | [worker creates file](../../mayan/apps/documents/tasks.py#L82-L89) | duplicate version | Medium | invoke twice; operation id/unique guard | Medium |
| Terminal async failure not visible to caller | 202 + worker logging [task](../../mayan/apps/documents/tasks.py#L103-L124) | poor UX/support | Medium | status-resource test | Medium |
| Blob/DB drift after partial delete | [multi-step delete](../../mayan/apps/documents/models/document_file_models.py#L215-L236) | orphan/missing bytes | Medium | failure injection + reconciliation | High |
| ACL regression from unscoped child query | [current scoping](../../mayan/apps/documents/api_views/document_file_api_views.py#L89-L120) | data exposure | Low/High impact | wrong-parent matrix | High |
| ACL query cost on large sets | [dynamic filters](../../mayan/apps/acls/managers.py#L268-L294) | latency | Unknown | benchmark/query plan | Low |
| OCR chord delayed by one poisoned page | [fan-in](../../mayan/apps/ocr/tasks.py#L30-L48) | stuck processing | Medium | transient/permanent page tests; partial state | Medium |
| Source lock expires before long processing | [lock acquisition](../../mayan/apps/sources/tasks.py#L24-L38) | duplicate ingestion | Unknown | duration>TTL concurrency test; lease/fencing | Low |
| Dynamic action args too permissive | [JSON/action dispatch](../../mayan/apps/sources/api_views.py#L42-L62) | backend misuse | Unknown | per-backend schema tests | Low |
| Toolchain ambiguity | `setup.py` vs `tox.ini` | contributor failure | High | maintainer-confirmed matrix | High |
