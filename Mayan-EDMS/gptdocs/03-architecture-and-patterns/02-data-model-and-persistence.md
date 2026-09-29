# Data model and persistence

The important distinction is Document (business identity) versus DocumentFile (bytes and derived metadata). The file has a foreign key, indexed timestamp/checksum/size, and defined storage at [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L74-L129). ACLs are generic object grants joining role and stored permissions at [acls/models.py](../../mayan/apps/acls/models.py#L22-L56).

Consistency spans SQL and blob storage. Deletion removes pages, blob, cache, then row at [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L215-L236); failures between steps need reconciliation.

Safe schema change: add nullable/default-compatible field → deploy readers tolerant of both shapes → backfill in bounded batches → add constraints/index concurrently where supported → switch writes → remove compatibility later. Test old/new rows, migration reversal policy, task payload compatibility, and rollback.
