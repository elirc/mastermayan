# Data Model and Persistence

## Key entities

| Entity | Relationship highlights | Evidence |
| --- | --- | --- |
| `DocumentType` | one-to-many to `Document`; unique label | [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L39-L79) |
| `Document` | belongs to `DocumentType`; has many `DocumentFile`; trash state inline | [`document_models.py`](../../mayan/apps/documents/models/document_models.py#L53-L104) |
| `DocumentFile` | belongs to `Document`; derived checksum/mimetype/size/pages | [`document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py#L74-L121) |
| `AccessControlList` | generic object reference + role + many permissions | [`acls/models.py`](../../mayan/apps/acls/models.py#L35-L58) |
| `DocumentCheckout` | one-to-one with `Document` | [`checkouts/models.py`](../../mayan/apps/checkouts/models.py#L32-L58) |

## Persistence expectations

- Document trashing is soft-delete-ish at first via `in_trash`, then hard delete later [`document_models.py`](../../mayan/apps/documents/models/document_models.py#L142-L163).
- `DocumentFile.save()` computes and persists several derived fields after initial insert [`document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py#L466-L484).
- ACL uniqueness is `(content_type, object_id, role)` [`acls/models.py`](../../mayan/apps/acls/models.py#L56-L60).

## Safely changing schema here

1. Inspect model plus app migrations.
2. Search for serializer, view, admin, search, and test dependencies.
3. Add a migration.
4. Decide whether upgrade tests should cover it.
5. Add forward behavior tests and, if risky, migration tests.

## What juniors often miss

- Some "state" is stored directly on main tables instead of separate event tables.
- Soft-delete flows require filtering discipline.
