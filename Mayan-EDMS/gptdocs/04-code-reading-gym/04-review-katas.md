# Review katas

## Kata 1: simplify download lookup
Fake diff: replace parent-scoped queryset with global file lookup. Resembles [download view](../../mayan/apps/documents/api_views/document_file_api_views.py#L93-L123). **Blocking:** IDOR risk. Comment: “Please retain parent scoping and add a wrong-parent regression test; object permission alone does not prove the URL relationship.”

## Kata 2: synchronous OCR
Fake diff: OCR every page in the request. Resembles [OCR chord](../../mayan/apps/ocr/tasks.py#L17-L48). **Blocking:** unbounded latency/timeouts. **Important:** capacity/backpressure plan.

## Kata 3: retry everything
Fake diff: `except Exception: self.retry()`. Resembles [upload task](../../mayan/apps/documents/tasks.py#L90-L124). **Blocking:** permanent errors loop. Request error taxonomy and max-attempt test.

## Kata 4: richer upload response
Fake diff: return the predicted file ID with 201 before worker completion. **Blocking:** contract claims a resource exists. Prefer 202 operation resource.

## Kata 5: faster checksum
Fake diff: `file.read()` once. Resembles [checksum loop](../../mayan/apps/documents/models/document_file_models.py#L186-L213). **Important:** memory scales with upload size; require benchmark.

## Kata 6: remove 404 disguise
Fake diff: return 403 for unauthorized file IDs. Tests currently expect 404 at [test_document_file_api.py](../../mayan/apps/documents/tests/test_document_file_api.py#L100-L106). **Important:** public/security contract change; document threat-model decision.

## Kata 7: cache authorized lists
Fake diff: cache list response only by URL. **Blocking:** identity-dependent ACL results may leak. Include authorization context or avoid shared cache.

## Kata 8: refactor task helpers
Fake diff: move all task logic into a new service layer without behavior tests. **Important:** high blast radius and hidden hook/event behavior. Ask for one vertical incremental change with characterization tests.

Grading: identify security/correctness blockers first, then maintainability/performance, then style. A good review is specific, evidence-linked, and proposes a verification path.
