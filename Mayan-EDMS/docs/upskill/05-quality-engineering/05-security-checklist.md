# Security Checklist

## Repo-specific risks

| Risk | Evidence | What to check before merge |
| --- | --- | --- |
| IDOR / missing object filter | ACL restriction pattern in [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L79-L86) | no raw PK fetch before auth |
| auth user enumeration | timing mitigation in [`django_authentication_backends.py`](../../mayan/apps/authentication/django_authentication_backends.py#L13-L20) | preserve constant-time-ish behavior |
| CSRF regression | middleware in [`mayan/settings/base.py`](../../mayan/settings/base.py#L108-L115) | do not bypass CSRF casually |
| upload abuse | archive expansion paths in [`document_models.py`](../../mayan/apps/documents/models/document_models.py#L205-L229) and [`sources/models.py`](../../mayan/apps/sources/models.py#L97-L124) | think about quotas, nesting, decompression load |
| permission drift in nested resources | cabinet/document split tests | verify both sides of relationship |

## Pre-merge checklist

- Did I preserve queryset-level permission filtering?
- Did I introduce a new public route?
- Did I move expensive or externally dependent work into a request path?
- Did I add or change any callback dotted paths or task signatures without tests?
