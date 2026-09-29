# Domain glossary

| Term | Meaning and anchor |
| --- | --- |
| Document | Business record; not identical to one byte stream. Files relate to it at [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L74-L77). |
| Document file | Uploaded binary plus checksum, size, MIME, timestamp: [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L78-L120). |
| Version/page | A presentation/order layer above file pages; OCR targets document-version pages: [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L51-L79). |
| Source | Configured ingestion backend with discoverable actions: [sources/serializers.py](../../mayan/apps/sources/serializers.py#L10-L46). |
| ACL | Actor-role + permission + object grant: [acls/models.py](../../mayan/apps/acls/models.py#L22-L49). |
| Workflow | Template/instance state machine; escalations are checked by scheduled tasks. |
| Shared uploaded file | Staging record that transfers request bytes to a worker: [documents/tasks.py](../../mayan/apps/documents/tasks.py#L60-L89). |
| Stub | A document temporarily lacking usable files; upload failure/deletion behavior must preserve its invariant. |

Near-synonyms to keep separate: authentication vs authorization; document vs file vs version; source action vs Celery task; checksum integrity vs cryptographic signature authenticity; global permission vs object ACL.
