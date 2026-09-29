# Performance thinking

Rule zero: **measure first.** This file lists where to *look*, with the evidence for why — not conclusions. The house already gives you one measuring stick: the test harness counts DB connections ([ConnectionsCheckTestCaseMixin, testing/tests/mixins.py L97](../../../mayan/apps/testing/tests/mixins.py)); add `assertNumQueries` to any view you touch and you have query-count regression tests for free.

## The performance domains of this app

| Domain | Hotspot candidates | Evidence / anchor |
| --- | --- | --- |
| DB: authz subqueries | every list view runs ACL Q-trees; inheritance recursion multiplies subqueries | [_get_acl_filters](../../../mayan/apps/acls/managers.py#L31-L231); `check_access` loops per permission ([L233–L266](../../../mayan/apps/acls/managers.py#L233-L266)) |
| DB: N+1 | serializer method fields, per-row `evaluate_condition` template renders ([workflow_instance_models.py L184–L188](../../../mayan/apps/document_states/models/workflow_instance_models.py#L171-L195)), `get_current_state` walking log entries per instance | fields are declared per-model via `ModelQueryFields` ([document_views.py L75–L78](../../../mayan/apps/documents/views/document_views.py#L75-L78)) — the select/prefetch hook exists; use it |
| Server: file hashing | SHA-256 streams the whole file per upload; block size configurable ([document_file_models.py L186–L213](../../../mayan/apps/documents/models/document_file_models.py#L186-L213)) | CPU on worker_c; fine until GB-scale files |
| Converter/OCR | per-page rendering + tesseract; entire office docs converted to PDF once and cached ([get_intermediate_file, L296–L334](../../../mayan/apps/documents/models/document_file_models.py#L296-L334)) | worker_d saturation; cache sizing (scenario 4 in [03-debugging](03-systematic-debugging.md)) |
| Cache math | `prune()` recomputes `get_total_size()` (a SUM aggregate) **per loop iteration** ([file_caching/models.py L114–L152](../../../mayan/apps/file_caching/models.py#L105-L152)) | O(files × prune steps) SUMs under lock — measure before touching, but it's a classic |
| Search | write amplification: every save → index task; related saves fan out | [handlers](../../../mayan/apps/dynamic_search/handlers.py#L56-L125); full reindex is chunked properly ([tasks L153–L165](../../../mayan/apps/dynamic_search/tasks.py#L153-L165)) |
| Frontend | server-rendered; page weight from vendored JS; no bundle to optimize | lazy-load lib present ([appearance/dependencies.py](../../../mayan/apps/appearance/dependencies.py#L46-L47)) |
| Memory | `block_size=0` sentinel reads whole file into memory per hash read call ([L191–L197](../../../mayan/apps/documents/models/document_file_models.py#L186-L213)); workers restart on memory caps ([workers.py](../../../mayan/apps/task_manager/workers.py#L12-L36)) | caps are the symptom-manager; find leaks before raising caps |

## How to find each class (repo-appropriate technique)

- **N+1**: `assertNumQueries` around list views with 1 vs 10 objects — count must not scale with rows. Or `connection.queries` in shell after a request.
- **Slow SQL**: take the ACL queryset in shell, `print(qs.query)`, run `EXPLAIN ANALYZE` in Postgres. The interesting plan: `id IN (subquery)` + OR across inheritance branches.
- **Serial async**: look for loops awaiting one task at a time — here the anti-example is already good (chord in OCR, fan-out in trash empty); cite them as the fix pattern.
- **Missing indexes**: hot filters are indexed ([02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md)); the ACL table's `(content_type, object_id)` pair is the one to verify with `EXPLAIN` under volume.
- **Unbounded queries**: grep for `.all()` iterated without pagination in tasks — trash empty fans out per row ([documents/tasks.py L299–L311](../../../mayan/apps/documents/tasks.py#L299-L311)), good; anything materializing `list(queryset)` of documents deserves a look ([pages_append_all does exactly that, document_version_models.py L299–L304](../../../mayan/apps/documents/models/document_version_models.py#L291-L309) — bounded by pages-per-document, so acceptable).

## The mid-level answer template (memorize)

"I'd reproduce with realistic volume, measure (query count / EXPLAIN / worker queue latency), fix the biggest term, and **add the measurement as a regression test**." Then give one concrete example — the ACL list view with `assertNumQueries` is your best story from this repo.

**Drill:** predict the query count for `GET /api/v4/documents/` returning 10 documents for a non-staff user with type-level ACLs. Count: 1 (page) + ACL subqueries (evaluated as part of the main query — subqueries don't add round trips!) + serializer joins (document_type, latest file, active version per row?). Now check `DocumentSerializer`'s nested fields ([document_serializers.py L23–L41](../../../mayan/apps/documents/serializers/document_serializers.py#L23-L41)) and find which nested serializer probably costs a query per row (`file_latest` — a per-instance `order_by().last()` property, [document_models.py L185–L187](../../../mayan/apps/documents/models/document_models.py#L185-L187)). *Strong:* you propose the `Prefetch`/annotation fix and the test that pins it.

**Interview angle:** performance rounds want *method* not folklore. The domain table + answer template is the method. Card [08/03 Q17](../08-interview-prep/03-api-and-data-modeling-questions.md).
