# Validation, Auth, and Permissions

## Validation layers

| Layer | Example | What it protects |
| --- | --- | --- |
| Form/serializer validation | upload serializers and source action serializers | request shape |
| Model validation | [`checkouts/models.py`](../../mayan/apps/checkouts/models.py#L68-L72) | semantic invariant |
| backend config validation | YAML validator in [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L70-L76) | plugin/config correctness |

## Authentication

- Email login backend with timing-attack mitigation:
  [`django_authentication_backends.py`](../../mayan/apps/authentication/django_authentication_backends.py#L13-L20)
- MFA wizard/session handoff:
  [`authentication_views.py`](../../mayan/apps/authentication/views/authentication_views.py#L189-L213)
- Global login-required posture:
  [`mayan/settings/base.py`](../../mayan/settings/base.py#L108-L123)

## Authorization

- Method-level permission declarations on API views:
  [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L32-L37)
- Queryset restriction by ACL:
  [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L79-L86)
- Object-level auth storage:
  [`acls/models.py`](../../mayan/apps/acls/models.py#L22-L58)

## What a junior might miss

- Primary key possession is not authorization.
- Permission checks often need both source access and target object access.

## What a senior checks

- Is the queryset filtered before object resolution?
- Are nested resources enforcing both parent and child access?
- Is any async callback operating on data that was authorized earlier but may have changed?
