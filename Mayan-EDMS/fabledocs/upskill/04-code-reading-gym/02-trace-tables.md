# Trace tables

Fill in each table yourself first (cover the answers). Columns: step, file:lines, value shape at that point, owner (layer), transformation applied, risk introduced.

## Trace 1 (UI → API-shaped): document label from upload form to database

| Step | File | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | client multipart | `file` part, `label` text field | browser | — | filename encoding/UTF-8 |
| 2 | [document_serializers.py L71–L76](../../../mayan/apps/documents/serializers/document_serializers.py#L71-L90) | `validated_data` dict; label defaulted to `str(file)` | serializer | `validated_data['label'] = validated_data.get('label', str(file))` | filename as label — user-controlled string |
| 3 | [documents/tasks.py L52–L88](../../../mayan/apps/documents/tasks.py#L52-L88) | kwargs of ints/strings | queue | label not passed! re-derived: `filename or shared_uploaded_file.filename` | divergence between step-2 label and worker filename |
| 4 | [document_models.py L230–L234](../../../mayan/apps/documents/models/document_models.py#L230-L234) | `DocumentFile.filename` | model | `Path(file_object.name).name` strips directories | path traversal neutralized here — name the exact line in review |
| 5 | [document_file_models.py L479–L484](../../../mayan/apps/documents/models/document_file_models.py#L479-L484) | `document.label` | model | if label empty → `force_text(self.file)` | label ≤255 chars enforced only by DB — long filenames? (max_length=255 both, [document_models.py L64–L71](../../../mayan/apps/documents/models/document_models.py#L64-L71)) |
| 6 | Postgres | row | DB | — | display escaping is the template's job (XSS boundary) |

## Trace 2 (persistence): checksum lifecycle

| Step | File | Value shape | Transformation | Risk |
| --- | --- | --- | --- | --- |
| 1 | [document_file_models.py L110–L116](../../../mayan/apps/documents/models/document_file_models.py#L110-L121) | `checksum` = NULL | column definition, indexed | |
| 2 | [L466–L471](../../../mayan/apps/documents/models/document_file_models.py#L466-L472) | still NULL | save → transaction begins | window where row exists w/o checksum (inside txn) |
| 3 | [L199–L209](../../../mayan/apps/documents/models/document_file_models.py#L186-L213) | hex string | streamed SHA-256, block size from setting | block_size=0 sentinel → `-1` read-all — memory spike on huge files |
| 4 | [duplicate_backends.py L15–L28](../../../mayan/apps/duplicates/duplicate_backends.py#L8-L28) | queryset | dedupe by *latest file's* checksum | annotate+filter Max trick — verify it matches "latest" semantics (`file_latest` = last by timestamp, [document_models.py L185–L187](../../../mayan/apps/documents/models/document_models.py#L185-L187)) |
| 5 | search ([documents/search.py L47–L49](../../../mayan/apps/documents/search.py#L44-L56)) | indexed term | `files__checksum` searchable | index staleness window (Flow 7) |

## Trace 3 (auth): one GET through the permission machine

| Step | File | Value shape | Risk |
| --- | --- | --- | --- |
| 1 | `GET /api/v4/documents/55/` + token | request.user | token theft = user |
| 2 | [document_api_views.py L31–L39](../../../mayan/apps/documents/api_views/document_api_views.py#L23-L44) | `mayan_object_permissions['GET']` → 1 permission object | wrong map entry = wrong policy |
| 3 | [rest_api/filters.py L7–L16](../../../mayan/apps/rest_api/filters.py#L6-L24) | queryset (unevaluated) | filter must run before `get_object` |
| 4 | [acls/managers.py L269–L277](../../../mayan/apps/acls/managers.py#L268-L294) | whole qs (role grant) or… | staff bypass breadth |
| 5 | [acls/managers.py L156–L194](../../../mayan/apps/acls/managers.py#L156-L194) | `Q(id__in=ACL-subquery) OR Q(document_type_id__in=…)` | subquery cost; inheritance recursion |
| 6 | DRF `get_object` | instance or 404 | 404 masks "exists but denied" (by design) |
| 7 | serializer | JSON incl. hyperlinks | leaking related URLs the user can't follow is fine (they 404) — know why |

## Trace 4 (error path): OCR failure on page 3 of 5

| Step | File | What happens | Residue |
| --- | --- | --- | --- |
| 1 | [ocr/tasks.py L31–L43](../../../mayan/apps/ocr/tasks.py#L17-L49) | chord: 5 page tasks + finisher | |
| 2 | pages 1,2,4,5 | content rows written | partial OCR searchable? (depends on indexing trigger) |
| 3 | page 3 [L80–L90](../../../mayan/apps/ocr/tasks.py#L74-L90) | cache miss → retry ×N | retry storm risk |
| 4 | retries exhausted | task fails; **chord never completes** | finisher ([L92–L117](../../../mayan/apps/ocr/tasks.py#L92-L117)) never runs → error log never cleared, finished event never fires |
| 5 | user | sees stale "error" state, no notification | the observability gap to name in interviews |

## Trace 5 (async fan-out): tag added to 3 documents → search freshness

| Step | File | What happens |
| --- | --- | --- |
| 1 | tag M2M change | `m2m_changed` signal → [handlers L84–L113](../../../mayan/apps/dynamic_search/handlers.py#L84-L113) |
| 2 | serialization | model classes → `"app.model"` strings, pk_set → tuple ([L98–L107](../../../mayan/apps/dynamic_search/handlers.py#L84-L113)) — queue contract again |
| 3 | [tasks L124–L150](../../../mayan/apps/dynamic_search/tasks.py#L118-L150) | strings → models via `apps.get_model`; backend re-indexes affected docs |
| 4 | searcher | sees new tag after queue drain — **eventual**; a test asserting immediately will flake ([05/01](../05-quality-engineering/01-testing-strategy.md)) |

**Self-grade across all traces:** *Basic* — you can fill file+transformation columns. *Solid* — your risk column matches ≥70% of the above. *Strong* — for one trace you extended it one step further than the table (e.g. Trace 2 step 6: what happens to the checksum when a file is *deleted* — nothing references it; dedupe entries are cleaned by [duplicates handlers](../../../mayan/apps/duplicates/handlers.py)).
