# Django and DRF mental models

Request routing selects a class-based view; serializers validate/shape data; permissions run at view and object boundaries; querysets delay database execution; models/managers contain persistence behavior. In upload, the serializer is validated before staging, then `get_document(permission=...)` scopes authorization before enqueue at [document_file_api_views.py](../../mayan/apps/documents/api_views/document_file_api_views.py#L38-L60).

Server-rendered Django UI has no React render/commit lifecycle. State is primarily request, session, form, URL, and database state; browser JavaScript is enhancement. Common sharp edges are N+1 lazy relations, fat models, hidden signals/hooks, and authorization performed after lookup.

Drill: trace one GET and one POST. Strong answers name when the queryset evaluates and where a transaction begins/ends.

## Interview angle

Use [frontend/framework questions](../08-interview-prep/02-frontend-framework-questions.md), especially request lifecycle, forms versus serializers, progressive enhancement, caching, and accessibility.
