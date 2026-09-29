# System map

## Shape: one Django project, 57 self-registering apps

This is **not** a monorepo of services. It is a single deployable Django 3.2 project ([mayan/settings/base.py](../../../mayan/settings/base.py#L42-L50) builds `INSTALLED_APPS`) whose functionality is split into apps under [mayan/apps/](../../../mayan/apps/). Each app is a vertical slice: models, UI views, API views, serializers, Celery tasks, permissions, events, and tests for one capability (tags, cabinets, checkouts, OCR…).

The composition trick: [mayan/urls/base.py](../../../mayan/urls/base.py) is an **empty list**. At startup, every app's `MayanAppConfig.ready()` calls `configure_urls()`, which imports the app's `urls.py` and appends it to the global urlconf ([common/apps.py](../../../mayan/apps/common/apps.py#L27-L120)). Permissions, events, queues, and search fields register through the same "import-time registry" idiom. **Senior noticing:** this buys drop-in modularity and costs you grep-ability — nothing references `documents.urls` explicitly, so you must know the convention to find the wiring.

```
Browser ──HTTP──> Django (gunicorn)                 ┌────────────────┐
   │                ├─ UI views (server-rendered)    │  RabbitMQ      │
   REST client ─────┤─ API views (DRF, /api/v4/)  ──>│  (broker)      │
                    │        │                       └──────┬─────────┘
                    │   ACL restrict_queryset               │ tasks by ID
                    ▼        ▼                       ┌──────▼─────────┐
              PostgreSQL (metadata, ACLs,            │ Celery workers │
              workflows, audit events)               │  A  B  C  D    │
                    ▲                                └──────┬─────────┘
                    │                                       │
              Object storage (document files) <─────────────┘
              Redis (locks, cache, result backend)
```

Broker/lock/result wiring is visible in [docker/docker-compose.yml](../../../docker/docker-compose.yml#L3-L16).

## Ownership map

| Concern | Owner apps | Entry points |
| --- | --- | --- |
| Core domain (documents, files, versions, pages, types, trash) | `documents` | [models/](../../../mayan/apps/documents/models/) split per entity |
| AuthN | `authentication`, `authentication_otp` | login views, token auth via [rest_api/urls.py](../../../mayan/apps/rest_api/urls.py#L12-L16) |
| AuthZ | `permissions` (role→permission), `acls` (per-object grants) | [acls/managers.py](../../../mayan/apps/acls/managers.py#L26-L30) |
| Ingestion | `sources` (+ backends: web form, staging, watch folder, email, scanner) | [source_backends/](../../../mayan/apps/sources/source_backends/) |
| Content extraction | `converter`, `ocr`, `document_parsing`, `file_metadata` | Celery tasks per app |
| Organization | `tags`, `cabinets`, `metadata`, `document_indexing`, `linking`, `web_links` | each app's `models.py` |
| Process automation | `document_states` (workflows), `document_comments`, `checkouts` | [workflow_instance_models.py](../../../mayan/apps/document_states/models/workflow_instance_models.py#L33-L58) |
| Search | `dynamic_search` (+ Whoosh/ES backends), per-app `search.py` declarations | [documents/search.py](../../../mayan/apps/documents/search.py#L22-L36) |
| Async plumbing | `task_manager` (queues/workers), `lock_manager` (distributed locks) | [task_manager/workers.py](../../../mayan/apps/task_manager/workers.py#L12-L36) |
| Files at rest | `storage` (DefinedStorage, SharedUploadedFile, DownloadFile), `file_caching` | [file_caching/models.py](../../../mayan/apps/file_caching/models.py#L38-L48) |
| Audit trail | `events` (django-activity-stream wrapper) | [events/classes.py](../../../mayan/apps/events/classes.py#L323-L446) |
| UI chrome | `appearance` (templates/Bootstrap), `navigation` (menus/links), `views` (generic view library) | [views/mixins.py](../../../mayan/apps/views/mixins.py) |
| Platform | `smart_settings` (env-driven settings), `dependencies`, `databases`, `logging`, `organizations`, `quotas` | [settings/base.py](../../../mayan/settings/base.py#L18-L28) |

## Public interfaces vs private internals

**Public (contracts — changing these breaks users):**
- REST API under `/api/v4/` — versioned in the URL ([rest_api/urls.py](../../../mayan/apps/rest_api/urls.py#L27-L34)), self-documented via Swagger/ReDoc ([rest_api/urls.py](../../../mayan/apps/rest_api/urls.py#L37-L45)).
- UI URL names (`documents:document_preview` etc.) — bookmarked by users, referenced by `get_absolute_url()` ([document_models.py](../../../mayan/apps/documents/models/document_models.py#L253-L258)).
- Environment variables `MAYAN_*` consumed by smart_settings ([settings/base.py](../../../mayan/settings/base.py#L30-L38)).
- Event type IDs (`documents.document_created`) — stored in the DB and in subscriptions; renaming one orphans history ([events/classes.py](../../../mayan/apps/events/classes.py#L443-L445)).

**Private (free to refactor):** manager internals, task function signatures (mostly — they're also contracts for in-flight queued messages during a deploy!), template block structure, the `views` generic-view library. **Senior noticing:** Celery task kwargs are a *hidden* public contract across deploy boundaries — a queued `task_document_file_upload` message from the old release must still be consumable by the new release ([documents/tasks.py](../../../mayan/apps/documents/tasks.py#L52-L55)).

## Drill

Without opening them, predict what lives in `mayan/apps/cabinets/` — file by file. Then open the app and score yourself. *Basic:* you predicted `models.py`, `views.py`, `urls.py`. *Solid:* you also predicted `permissions.py`, `events.py`, `serializers.py`, `api_views.py`, and `search.py`. *Strong:* you predicted the ACL registration in `apps.py` and can say **why** the registry idiom forces every capability to have an `apps.py` wiring block.

**Interview angle:** "How is the codebase organized?" is a warm-up in most technical deep-dives. A mid-level answer names the skeleton and the registration idiom; a senior answer adds the tradeoff (modularity vs discoverability) and where the hidden contracts are. Cross-reference: [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).
