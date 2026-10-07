# Trace Tables

## UI to API trace

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| user uploads file | [`web_form_backends.py`](../../../mayan/apps/sources/source_backends/web_form_backends.py#L42-L70) | multipart file + `document_type_id` | source backend | ACL filter, temp upload creation | unauthorized type |
| task reload | [`sources/tasks.py`](../../../mayan/apps/sources/tasks.py#L64-L87) | DB ids | upload worker | rehydrate models | stale/missing rows |
| document create | [`document_type_models.py`](../../../mayan/apps/documents/models/document_type_models.py#L146-L176) | label, description, file object | document type | document then file | partial create |

## Persistence trace

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| new file row | [`document_file_models.py`](../../../mayan/apps/documents/models/document_file_models.py#L231-L236) | file storage handle | model | insert | storage failure |
| derived state | [`document_file_models.py`](../../../mayan/apps/documents/models/document_file_models.py#L466-L484) | checksum, mime, size, pages | model | enrich | slow save |

## Auth trace

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| incoming credentials | [`django_authentication_backends.py`](../../../mayan/apps/authentication/django_authentication_backends.py#L8-L21) | username/password | auth backend | email lookup, password check | enumeration |
| MFA session handoff | [`authentication_views.py`](../../../mayan/apps/authentication/views/authentication_views.py#L197-L213) | user pk in session | login view | redirect to wizard | stale session |

## Error trace

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| OCR page task error | [`ocr/tasks.py`](../../../mayan/apps/ocr/tasks.py#L75-L89) | exception | page task | retry | endless retries |
| OCR finish error | [`ocr/tasks.py`](../../../mayan/apps/ocr/tasks.py#L114-L125) | exception | finish task | error log entry | completion ambiguity |
