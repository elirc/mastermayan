# Boundaries and Layers

## Layer map

| Layer | Owns | Must not own | Example |
| --- | --- | --- | --- |
| UI/browser | interaction, rendering, AJAX navigation | core auth or persistence rules | [`partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L87-L152) |
| Transport/API | request contract, serializer validation, permission mapping | heavy business orchestration | [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L64-L135) |
| Domain/model | invariants, lifecycle, orchestration of core records | low-level HTTP concerns | [`document_models.py`](../../mayan/apps/documents/models/document_models.py#L142-L184) |
| Async worker | retryable side effects, expensive enrichment | browser-specific behavior | [`ocr/tasks.py`](../../mayan/apps/ocr/tasks.py#L17-L131) |
| Persistence | durable storage and migrations | UI assumptions | app migrations and models |
| Security boundary | ACL restriction, auth backend, middleware | ad hoc post-fetch checks only | [`acls/models.py`](../../mayan/apps/acls/models.py#L22-L58) |

## Good boundaries

- Request-side upload permission check before queuing [`web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L42-L50)
- Task-side object reloading by PK [`sources/tasks.py`](../../mayan/apps/sources/tasks.py#L64-L87)
- Domain rollback when first file creation fails [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L163-L174)

## Boundary leaks

- Signal-driven cross-app work can hide ownership.
- Model `save()` methods perform expensive derived work.
- Client navigation logic implicitly depends on backend partial-render behavior.

## Senior noticing

- This codebase favors pragmatic in-model orchestration over formal service layers.
- That works, but only if reviewers remain disciplined about side-effect sprawl.
