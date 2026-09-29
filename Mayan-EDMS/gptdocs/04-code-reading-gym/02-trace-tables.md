# Trace tables

Complete the blanks before checking source.

## UI/API to worker upload

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| validate | [view](../../mayan/apps/documents/api_views/document_file_api_views.py#L38-L42) | multipart → validated dict | API | schema validation | size/content |
| stage | lines 44–47 | file → staging ID | storage | durable handoff | orphan |
| enqueue | lines 49–60 | IDs/scalars | broker | serialize | enqueue failure |
| create | [task](../../mayan/apps/documents/tasks.py#L82-L89) | file object → domain call | worker | durable document file | duplicate |

## Authorization trace

| Step | File/line | Value | Owner | Risk |
| --- | --- | --- | --- | --- |
| declare | [view](../../mayan/apps/documents/api_views/document_file_api_views.py#L53-L55) | permission object | endpoint | omission |
| adapt | [permission](../../mayan/apps/rest_api/permissions.py#L26-L42) | request/view/object | DRF | default allow |
| restrict | [ACL](../../mayan/apps/acls/managers.py#L233-L266) | authorized queryset | ACL | inheritance complexity |
| assert | [test](../../mayan/apps/documents/tests/test_document_file_api.py#L100-L129) | 404/200 | test | missing cross-parent case |

## Persistence deletion trace

Pages → blob → cache → row → possible stub update: [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L215-L236). Add a column for compensation after each step.

## OCR error trace

Chord creation → page image/cache lookup → OCR persistence → retry → finish event/error record: [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L17-L125). Mark which failures are visible to a user.

## Source polling trace

Task → lock → source lookup → backend → error log → release: [sources/tasks.py](../../mayan/apps/sources/tasks.py#L18-L55). Mark lock owner and expiry assumption.
