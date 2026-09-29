# Data model and persistence

## Core entity graph

```
DocumentType 1──* Document 1──* DocumentFile 1──* DocumentFilePage
                     │                                   ▲
                     └────1──* DocumentVersion 1──* DocumentVersionPage
                                                  (GenericFK content_object ──┘ usually)
Role *──* Group *──* User          AccessControlList (content_type, object_id, role) *──* StoredPermission
Workflow 1──* WorkflowState/Transition ; WorkflowInstance (document, workflow) 1──* WorkflowInstanceLogEntry
Cache 1──* CachePartition 1──* CachePartitionFile
```

Key declarations: Document ([document_models.py L41–L108](../../../mayan/apps/documents/models/document_models.py#L41-L108)), DocumentFile ([document_file_models.py L51–L129](../../../mayan/apps/documents/models/document_file_models.py#L51-L129)), DocumentVersion ([document_version_models.py L45–L72](../../../mayan/apps/documents/models/document_version_models.py#L45-L72)), WorkflowInstance with `unique_together('document','workflow')` ([workflow_instance_models.py L54–L58](../../../mayan/apps/document_states/models/workflow_instance_models.py#L54-L58)), ACL rows ([acls/models.py](../../../mayan/apps/acls/models.py)).

The design insight worth repeating in interviews: **files are immutable facts; versions are mutable presentations.** New uploads append `DocumentFile`s; `DocumentVersionPage` rows *point* at file pages via GenericFK and can be remapped/reordered/deleted without touching bytes ([pages_remap, document_version_models.py L315–L338](../../../mayan/apps/documents/models/document_version_models.py#L315-L338)). That separation gives audit-grade history for free and makes "undo" a metadata operation.

## Indexes and lookup paths

Hot fields carry `db_index=True`: `Document.label`, `datetime_created`, `in_trash`, `is_stub` ([document_models.py L64–L104](../../../mayan/apps/documents/models/document_models.py#L64-L104)); `DocumentFile.checksum`, `size`, `timestamp` ([document_file_models.py L78–L121](../../../mayan/apps/documents/models/document_file_models.py#L78-L121)); `CachePartitionFile.hits` for prune ordering ([file_caching/models.py L341–L345](../../../mayan/apps/file_caching/models.py#L327-L351)). Missing-by-design: no composite index for the ACL lookup `(content_type, object_id, role)` beyond what the FKs give — a place to *measure before optimizing* ([05/04 performance](../05-quality-engineering/04-performance-thinking.md)).

## Transactions: where atomicity is actually claimed

The repo uses `transaction.atomic()` sparingly and locally:

- Derived-field cluster in `DocumentFile.save()` ([document_file_models.py L466–L484](../../../mayan/apps/documents/models/document_file_models.py#L466-L484)) — checksum/mime/size/pages/parent-stub update commit together. **Invariant protected:** no document file visible with half-computed metadata.
- Version activation: deactivate siblings + activate self ([document_version_models.py L95–L101](../../../mayan/apps/documents/models/document_version_models.py#L95-L101) inside `save`, [L373–L378](../../../mayan/apps/documents/models/document_version_models.py#L373-L378)). **Invariant:** ≤1 active version per document. Note it's enforced by *transactional convention*, not a DB constraint — a partial unique index would make the DB the enforcer. Good design-kata material.
- Trash flip ([document_models.py L149–L151](../../../mayan/apps/documents/models/document_models.py#L146-L163)) and hard-delete cascade ([L155–L159](../../../mayan/apps/documents/models/document_models.py#L155-L159)).

What is **not** transactional, deliberately or by omission: storage writes vs DB rows (bytes land when the `FileField` streams; a rollback strands them — Flow 4), event/audit rows (`event_document_file_created` commits before the derived-field transaction, [L460–L464](../../../mayan/apps/documents/models/document_file_models.py#L460-L464)), and anything crossing the queue. **Consistency expectation:** DB is source of truth; storage and search are eventually consistent with reapers/rebuilds as the repair path.

## Concurrency control

Three mechanisms, escalating in cost:
1. **DB constraints** as last line: checkout `OneToOneField` makes double-checkout impossible even when the check races ([checkouts/models.py L32–L35](../../../mayan/apps/checkouts/models.py#L28-L58), race at [L100–L109](../../../mayan/apps/checkouts/models.py#L100-L116)).
2. **Distributed locks** for cross-process critical sections: cache file create/delete ([file_caching/models.py L225–L272](../../../mayan/apps/file_caching/models.py#L225-L272)), one-at-a-time source processing ([sources/tasks.py L24–L33](../../../mayan/apps/sources/tasks.py#L24-L33)).
3. **`F()` expressions** for lock-free counters: `hits=F('hits')+1` ([file_caching/models.py L415](../../../mayan/apps/file_caching/models.py#L406-L416)) — the read-modify-write race eliminated in SQL.

Not used anywhere relevant: `select_for_update` (the workflow transition race in Flow 6 is the consequence).

## How to change the schema safely here

1. Model change in the owning app; run `makemigrations` (`inferred`) — never hand-edit an applied migration.
2. Additive first: new fields nullable/defaulted so old code and **in-flight Celery messages** stay valid across the deploy window.
3. Data migrations as separate `RunPython` steps; several apps keep `setting_migrations.py` for config schema evolution too ([documents/setting_migrations.py](../../../mayan/apps/documents/setting_migrations.py)).
4. Write a migrator test (`MayanMigratorTestCase`, [testing/tests/base.py L83–L88](../../../mayan/apps/testing/tests/base.py#L83-L88); example: [documents/tests/test_migrations.py](../../../mayan/apps/documents/tests/test_migrations.py)).
5. Renames/drops are two-release operations (add+backfill+dual-read, then drop). **Rollback plan:** migrations here are mostly reversible; anything with `RunPython` needs an explicit reverse function or it blocks `migrate` backwards.

**Drill (trace it):** a document has files F1 (3 pages) and F2 (2 pages); the active version maps [F2p1, F2p2]. The user runs the "append all file pages" modification. Using [pages_append_all L291–L309](../../../mayan/apps/documents/models/document_version_models.py#L291-L309), write the expected final mapping — then look at the `order_by` on line 300–301. *Investigate:* `document_file__page_number` is not a field of `DocumentFile` ([DocumentFile fields, document_file_models.py L74–L121](../../../mayan/apps/documents/models/document_file_models.py#L74-L121); `page_number` lives on `DocumentFilePage`, [document_file_page_models.py L38](../../../mayan/apps/documents/models/document_file_page_models.py#L38)) — Django should raise `FieldError` when this queryset evaluates, inside a fire-and-forget task ([documents/tasks.py L219–L233](../../../mayan/apps/documents/tasks.py#L218-L233)) where the user sees nothing. *Basic:* you compute the intended [F1p1..F2p2] mapping. *Solid:* you spot the suspect ordering path. *Strong:* you write the regression test that would have caught it (call `pages_append_all` directly in a model test) and the correct ordering (`'document_file__timestamp', 'page_number'`).

**Interview angle:** immutable-facts-vs-mutable-views, "what belongs in a transaction," and the three-tier concurrency toolkit are all top-frequency questions. Cards [08/03 Q9, Q13–Q15](../08-interview-prep/03-api-and-data-modeling-questions.md).
