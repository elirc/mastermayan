# System Map

## Repo shape

Mayan EDMS is a single Python package with many internal Django apps, not a JS monorepo and not a microservice fleet. The root runtime is assembled in [`mayan/settings/base.py`](../../mayan/settings/base.py#L34-L123), with app-level URL registration and local app boundaries under `mayan/apps`.

```text
Mayan-EDMS/
|- manage.py
|- Makefile
|- docker/
|- docs/
|- mayan/
|  |- settings/
|  |- urls/
|  |- celery.py
|  `- apps/
|     |- documents/
|     |- sources/
|     |- acls/
|     |- permissions/
|     |- authentication/
|     |- rest_api/
|     |- ocr/
|     |- document_parsing/
|     |- file_metadata/
|     |- checkouts/
|     |- cabinets/
|     `- ...
`- requirements/
```

## Runtime map

| Surface | Owner | Evidence | Notes |
| --- | --- | --- | --- |
| Django web app | `mayan/settings`, app `views.py`, app `urls.py` | [`mayan/settings/base.py`](../../mayan/settings/base.py#L34-L123), [`mayan/apps/documents/urls.py`](../../mayan/apps/documents/urls.py#L95-L520) | Main user-facing HTML surface. |
| REST API | `mayan/apps/rest_api` + app API views | [`mayan/apps/rest_api/urls.py`](../../mayan/apps/rest_api/urls.py#L10-L45), [`mayan/apps/documents/urls.py`](../../mayan/apps/documents/urls.py#L522-L688) | Versioned under `/api/v4/`. |
| Async workers | Celery + app tasks | [`mayan/celery.py`](../../mayan/celery.py#L1-L11), [`mayan/apps/sources/tasks.py`](../../mayan/apps/sources/tasks.py#L18-L103) | Upload, OCR, parsing, indexing, metadata. |
| Persistence | Django ORM, migrations | [`mayan/apps/documents/models/document_models.py`](../../mayan/apps/documents/models/document_models.py), migrations per app | Single DB, many app-local models. |
| UI JavaScript | jQuery-based static assets | [`mayan/apps/appearance/static/appearance/js/mayan_app.js`](../../mayan/apps/appearance/static/appearance/js/mayan_app.js), [`partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js) | Important but not dominant. |
| Deployment | Docker Compose | [`docker/docker-compose.yml`](../../docker/docker-compose.yml#L3-L289) | Multiple profiles for frontend, workers, infra. |

## Ownership map

| Area | Primary files |
| --- | --- |
| Core document domain | [`documents`](../../mayan/apps/documents/) |
| Ingestion/upload sources | [`sources`](../../mayan/apps/sources/) |
| ACL and roles | [`acls`](../../mayan/apps/acls/), [`permissions`](../../mayan/apps/permissions/) |
| Auth and login UX | [`authentication`](../../mayan/apps/authentication/), [`authentication_otp`](../../mayan/apps/authentication_otp/) |
| Async enrichment | [`file_metadata`](../../mayan/apps/file_metadata/), [`document_parsing`](../../mayan/apps/document_parsing/), [`ocr`](../../mayan/apps/ocr/) |
| Categorization | [`cabinets`](../../mayan/apps/cabinets/), [`tags`](../../mayan/apps/tags/), [`metadata`](../../mayan/apps/metadata/) |
| API framework glue | [`rest_api`](../../mayan/apps/rest_api/) |
| Locking/reliability | [`lock_manager`](../../mayan/apps/lock_manager/) |
| Tests | app-local `tests/` directories, GitLab CI config |

## Public interfaces vs private internals

### Public interfaces

- Browser routes in app `urls.py`, for example [`mayan/apps/documents/urls.py`](../../mayan/apps/documents/urls.py#L95-L520).
- API routes in `api_urls` lists, for example [`mayan/apps/documents/urls.py`](../../mayan/apps/documents/urls.py#L522-L688).
- Celery task signatures used across apps, for example [`mayan/apps/sources/tasks.py`](../../mayan/apps/sources/tasks.py#L58-L103).
- Backend extension points such as filename generators in [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L122-L133).

### Private internals

- Signal-driven coupling between apps.
- Hooks on document and document-file models.
- Error-log side channels in OCR and parsing apps.
- UI partial-navigation implementation details in JavaScript.

## Senior noticing

- This repo uses many Django apps as internal modules, but they are still in-process and share the same DB and event system. Boundaries are conceptual, not network-enforced.
- The true integration seams are permissions, model hooks, Celery tasks, and signals.
- Worker reliability depends on lock discipline and retry semantics more than on transaction boundaries alone.

## Verification Notes

- Inspected root inventory with `rg --files`.
- Inspected app list via `Get-ChildItem mayan/apps -Directory`.
- Runtime conclusions anchored to settings, Celery, Compose, and route files rather than README summaries alone.
