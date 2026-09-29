# Performance thinking

Measure before changing. Domains: template/render CPU, API payload/network, ORM query count/latency, storage throughput, OCR CPU, broker queue age, cache hit rate, and search latency.

Likely investigation targets, not confirmed problems: ACL query construction [managers.py](../../mayan/apps/acls/managers.py#L268-L294); page iteration/fan-out [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L30-L43); file hashing I/O [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L186-L213). Find N+1 with query capture and `select_related/prefetch_related`; serial async with timelines; unbounded lists with pagination review; missing indexes with query plans; worker pressure with queue age.

Require baseline, representative dataset, p50/p95/p99, resource cost, correctness guard, and rollback.
