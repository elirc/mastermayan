# File Reading Order

## Junior path

| File | Why it matters | What to look for | Ignore for now |
| --- | --- | --- | --- |
| [`README.md`](../../../README.md) | Product identity and install posture | Docker-first messaging | Marketing badges |
| [`CONTRIBUTING.md`](../../../CONTRIBUTING.md) | Contribution workflow | branch strategy, tests, sign-off | Translation details |
| [`Makefile`](../../../Makefile#L10-L551) | Real commands | test targets, staging, docs | release plumbing |
| [`mayan/settings/base.py`](../../../mayan/settings/base.py#L34-L123) | Runtime composition | `INSTALLED_APPS`, middleware, DRF, Celery | all language settings |
| [`mayan/celery.py`](../../../mayan/celery.py#L1-L11) | Worker bootstrap | app discovery | nothing |
| [`mayan/apps/documents/urls.py`](../../../mayan/apps/documents/urls.py#L95-L688) | Main surface map | HTML vs API routes | memorize every regex |
| [`mayan/apps/rest_api/urls.py`](../../../mayan/apps/rest_api/urls.py#L10-L45) | API root | versioning, schema docs | swagger details |
| [`mayan/apps/sources/source_backends/web_form_backends.py`](../../../mayan/apps/sources/source_backends/web_form_backends.py#L28-L72) | Simple ingest path | ACL check, staged upload, task queue | HTML template details |
| [`mayan/apps/sources/tasks.py`](../../../mayan/apps/sources/tasks.py#L58-L103) | Background upload | retry on operational error | source polling task |
| [`mayan/apps/documents/models/document_type_models.py`](../../../mayan/apps/documents/models/document_type_models.py#L138-L176) | document creation contract | rollback on failed first file | quick labels |
| [`mayan/apps/documents/models/document_models.py`](../../../mayan/apps/documents/models/document_models.py#L189-L251) | document-file addition | expand archives, actions | favorites, recent docs |
| [`mayan/apps/documents/models/document_file_models.py`](../../../mayan/apps/documents/models/document_file_models.py#L428-L505) | derived state on save | checksum, mimetype, page count, signals | cache internals |

## Mid-level path

Add these after the junior path:

| File | Why it matters | What to look for | Avoid distraction |
| --- | --- | --- | --- |
| [`mayan/apps/rest_api/api_view_mixins.py`](../../../mayan/apps/rest_api/api_view_mixins.py#L102-L172) | common API contracts | external object loading, serializer context | schema helper details |
| [`mayan/apps/acls/models.py`](../../../mayan/apps/acls/models.py#L22-L118) | fine-grained auth model | `content_type` + `object_id` + `role` | admin wiring |
| [`mayan/apps/documents/api_views/document_api_views.py`](../../../mayan/apps/documents/api_views/document_api_views.py#L23-L135) | API permission shape | `mayan_object_permissions`, ACL filtering | docstrings only |
| [`mayan/apps/file_metadata/methods.py`](../../../mayan/apps/file_metadata/methods.py#L7-L28) | async submission pattern | event + `user_id` + task | getters |
| [`mayan/apps/file_metadata/tasks.py`](../../../mayan/apps/file_metadata/tasks.py#L17-L48) | lock-based concurrency | one-file-at-a-time invariant | driver internals |
| [`mayan/apps/document_parsing/tasks.py`](../../../mayan/apps/document_parsing/tasks.py#L11-L34) | parsing worker | read-side enrichment | page-content models |
| [`mayan/apps/ocr/tasks.py`](../../../mayan/apps/ocr/tasks.py#L17-L131) | fan-out/fan-in async design | chord, retries, finish event | backend OCR drivers |
| [`mayan/apps/checkouts/models.py`](../../../mayan/apps/checkouts/models.py#L28-L116) | business invariant example | one checkout per document, future expiration | proxy model |
| [`mayan/apps/authentication/views/authentication_views.py`](../../../mayan/apps/authentication/views/authentication_views.py#L45-L145) | login flow | MFA wizard layering | password reset templates |
| [`docker/docker-compose.yml`](../../../docker/docker-compose.yml#L121-L230) | runtime topology | separate frontend, workers, beat | Traefik if not relevant |

## Senior path

Read these when you are evaluating design quality or preparing a proposal:

| File | What seniors look for |
| --- | --- |
| [`mayan/settings/base.py`](../../../mayan/settings/base.py#L34-L123) | cross-cutting commitments and legacy pressure |
| [`mayan/apps/sources/models.py`](../../../mayan/apps/sources/models.py#L87-L161) | callback design, task indirection, archive expansion risks |
| [`mayan/apps/document_indexing/handlers.py`](../../../mayan/apps/document_indexing/handlers.py#L39-L83) | signal-driven coupling and task storm potential |
| [`mayan/apps/appearance/static/appearance/js/partial_navigation.js`](../../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L87-L152) | client-side throttling, navigation ownership, error handling |
| [`mayan/apps/appearance/static/appearance/js/mayan_app.js`](../../../mayan/apps/appearance/static/appearance/js/mayan_app.js#L117-L197) | old-but-real front-end state/event orchestration |
| [`.gitlab-ci.yml`](../../../.gitlab-ci.yml#L249-L346) | test matrix, upgrade discipline, missing GitHub-native workflow |

## Pause and predict prompts

- Before opening `document_file_models.py`, predict which fields are persisted directly and which are derived at save time.
- Before reading `ocr/tasks.py`, predict where retries happen and what state is cleaned up on success.
- Before reading `partial_navigation.js`, predict what breaks if two rapid clicks produce overlapping AJAX responses.
