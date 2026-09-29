# Framework Mental Models

## Django mental model

Plain language: Django here is the composition root. Settings load apps, middleware gates requests, app `urls.py` files expose surfaces, and models carry meaningful domain logic.

Real examples:

- App composition in [`mayan/settings/base.py`](../../mayan/settings/base.py#L34-L123)
- Route composition in [`documents/urls.py`](../../mayan/apps/documents/urls.py#L95-L688)
- Login-required default in [`mayan/settings/base.py`](../../mayan/settings/base.py#L108-L123)

Use this mental model:

1. Request enters middleware.
2. URL resolves to a view.
3. View enforces permission and selects objects.
4. Models/managers perform domain work.
5. Signals/tasks fan out side effects.

Sharp edges:

- Business logic in models can surprise people expecting service classes.
- Signals create non-local behavior.
- Generic foreign keys in ACLs reduce compile-time discoverability.

## DRF mental model

Plain language: DRF views here are thin contract layers around queryset restriction, serializer validation, and object-action patterns.

Real examples:

- Permission mappings in [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L23-L135)
- common mixins in [`rest_api/api_view_mixins.py`](../../mayan/apps/rest_api/api_view_mixins.py#L102-L172)

Pitfalls:

- confusing `mayan_object_permissions` with queryset filtering
- forgetting external-object context when nested resources are involved

Drill:

- Find one API view that uses an external object and explain how serializer context gets the parent object.

## Celery mental model

Plain language: Celery is the repo’s boundary for expensive, retryable, or fan-out work.

Real examples:

- bootstrap in [`mayan/celery.py`](../../mayan/celery.py#L1-L11)
- retry-backoff upload task in [`sources/tasks.py`](../../mayan/apps/sources/tasks.py#L58-L103)
- page fan-out OCR in [`ocr/tasks.py`](../../mayan/apps/ocr/tasks.py#L30-L43)

What to watch:

- idempotency
- lock scope
- task storms from signals
- cleanup of temporary state
