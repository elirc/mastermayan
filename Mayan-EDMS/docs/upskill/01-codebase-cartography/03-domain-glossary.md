# Domain Glossary

| Term | Meaning here | Where to inspect |
| --- | --- | --- |
| Document | Logical record representing a managed business file set. Can start as a stub. | [`document_models.py`](../../mayan/apps/documents/models/document_models.py#L41-L105) |
| Document type | Class/category that controls policy and upload behavior. | [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L34-L79) |
| Document file | A concrete uploaded file associated with a document. | [`document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py#L51-L129) |
| Document version | Renderable/active version surface for a document. | [`documents/urls.py`](../../mayan/apps/documents/urls.py#L273-L343) |
| Source | Ingestion mechanism such as web form, watch folder, scanner, email. | [`sources/models.py`](../../mayan/apps/sources/models.py#L28-L39), [`sources/urls.py`](../../mayan/apps/sources/urls.py) |
| ACL | Object-level permission grant tying role, permission, and object. | [`acls/models.py`](../../mayan/apps/acls/models.py#L22-L58) |
| Role | System or object-level actor bucket used by ACLs and permissions. | [`permissions/models.py`](../../mayan/apps/permissions/models.py) |
| Stub document | DB row without an uploaded file yet. | [`document_models.py`](../../mayan/apps/documents/models/document_models.py#L98-L104) |
| Shared uploaded file | Temporary persisted upload handoff for async processing. | [`web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L57-L70), [`sources/tasks.py`](../../mayan/apps/sources/tasks.py#L68-L103) |
| Cabinet | Hierarchical container for documents. | [`cabinets/models.py`](../../mayan/apps/cabinets/models.py#L104-L104) |
| Quick label | Reusable document-type-specific filename/label option. | [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L193-L238) |
| Error log | Side-channel model attached by app config to track async failures. | [`document_parsing/apps.py`](../../mayan/apps/document_parsing/apps.py#L128-L129), [`ocr/apps.py`](../../mayan/apps/ocr/apps.py#L142-L143) |

## Confusing near-synonyms

- `Document` vs `DocumentFile`:
  The document is the durable business object; the file is one uploaded binary artifact attached to it.
- `Role` vs `ACL`:
  A role is an actor grouping; an ACL is a scoped permission grant on one object.
- `Document version` vs `Document file`:
  File is uploaded binary storage; version is the user-facing renderable/editable concept layered on top.
- `Source` vs `Source backend`:
  `Source` is the DB model/config object; source backend is the pluggable behavior class.
