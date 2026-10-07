# File reading order

Thirty files, ordered so each one makes the next legible. Per file: why it matters, what to look for, what to skip. Junior path = 1–14. Mid path = 1–24. Senior path = all, plus the "senior lens" notes.

## Phase 1 — orientation (everyone)

1. [mayan/__init__.py](../../../mayan/__init__.py) — v4.3.1, Django 3.2. *Look for:* `__build__` scheme. *Skip:* nothing, it's 12 lines.
2. [README.md](../../../README.md) — product identity. *Skip:* badge wall.
3. [Makefile](../../../Makefile#L1-L60) — how tests actually run (`manage.py test` + testing settings + `--skip-migrations` default). *Look for:* `MODULE=` targeting.
4. [mayan/settings/base.py](../../../mayan/settings/base.py#L18-L38) — the `SettingNamespaceSingleton` generating settings from env vars; the SECRET_KEY file fallback. *Senior lens:* settings-as-code-registry means `grep MAYAN_` beats reading this file.
5. [mayan/apps/common/apps.py](../../../mayan/apps/common/apps.py#L27-L120) — `MayanAppConfig.configure_urls`. The Rosetta stone for "where is anything wired?"
6. [mayan/celery.py](../../../mayan/celery.py) — 11 lines that create the Celery app and autodiscover tasks.

## Phase 2 — the core domain (everyone)

7. [documents/models/document_models.py](../../../mayan/apps/documents/models/document_models.py#L41-L108) — `Document`: three managers, `is_stub`, trash flags. *Look for:* `delete()` at [L142–L163](../../../mayan/apps/documents/models/document_models.py#L142-L163) — soft-delete-first.
8. [documents/models/document_file_models.py](../../../mayan/apps/documents/models/document_file_models.py#L428-L504) — `DocumentFile.save()` pipeline. *Look for:* what runs inside vs outside the `transaction.atomic()` block.
9. [documents/models/document_version_models.py](../../../mayan/apps/documents/models/document_version_models.py#L315-L338) — versions are *page mappings*, not file copies. `pages_remap` is the concept in one method.
10. [documents/managers.py](../../../mayan/apps/documents/managers.py) — what `valid` actually filters. *Predict before opening:* trash? stubs? both?
11. [documents/queues.py](../../../mayan/apps/documents/queues.py) — task registration, periodic schedules.
12. [documents/tasks.py](../../../mayan/apps/documents/tasks.py#L52-L124) — upload task with its retry-and-cleanup choreography.

## Phase 3 — authorization (everyone; re-read at every level)

13. [permissions/models.py](../../../mayan/apps/permissions/models.py#L193-L216) — `user_has_this`: superuser/staff bypass, then role→group membership.
14. [acls/managers.py](../../../mayan/apps/acls/managers.py#L268-L294) — `restrict_queryset`. Then the recursive filter builder at [L31–L231](../../../mayan/apps/acls/managers.py#L31-L231). *Junior:* read the 7 numbered cases in the comment. *Senior lens:* every case is a subquery — think about the SQL this emits on a 1M-document table.

## Phase 4 — request surfaces (mid+)

15. [rest_api/generics.py](../../../mayan/apps/rest_api/generics.py#L20-L72) — DRF base classes with `MayanObjectPermissionsFilter` baked in.
16. [rest_api/filters.py](../../../mayan/apps/rest_api/filters.py#L6-L24) — the filter itself; note permission lookup per HTTP method.
17. [documents/api_views/document_api_views.py](../../../mayan/apps/documents/api_views/document_api_views.py#L23-L44) — a complete, typical API view: `mayan_object_permissions` dict + `Document.valid` queryset.
18. [documents/serializers/document_serializers.py](../../../mayan/apps/documents/serializers/document_serializers.py#L71-L99) — `DocumentUploadSerializer.create` dispatching a Celery task from inside a serializer. *Senior lens:* is a serializer the right owner for that side effect?
19. [views/mixins.py](../../../mayan/apps/views/mixins.py#L549-L588) — `RestrictedQuerysetViewMixin`, the UI twin of the API filter.
20. [documents/views/document_views.py](../../../mayan/apps/documents/views/document_views.py#L36-L78) — `DocumentListView` and its defensive `get_context_data`.

## Phase 5 — async and integrity (mid+)

21. [events/decorators.py](../../../mayan/apps/events/decorators.py#L8-L33) + [events/classes.py](../../../mayan/apps/events/classes.py#L199-L216) — `@method_event` and `EventManagerSave`.
22. [sources/source_backends/mixins.py](../../../mayan/apps/sources/source_backends/mixins.py#L61-L120) — `process_documents`: files handed to workers **by SharedUploadedFile ID**, never by bytes.
23. [lock_manager/backends/file_lock.py](../../../mayan/apps/lock_manager/backends/file_lock.py#L62-L92) — a whole distributed-locks course in 116 lines (and its sharp edges).
24. [file_caching/models.py](../../../mayan/apps/file_caching/models.py#L225-L272) — `CachePartition.create_file`: lock → prune → write → size check.

## Phase 6 — senior sweep

25. [document_states/models/workflow_instance_models.py](../../../mayan/apps/document_states/models/workflow_instance_models.py#L82-L104) — `do_transition` and its exception swallowing; state as last-log-entry at [L132–L143](../../../mayan/apps/document_states/models/workflow_instance_models.py#L132-L143).
26. [ocr/tasks.py](../../../mayan/apps/ocr/tasks.py#L17-L49) — Celery chord fan-out/fan-in.
27. [dynamic_search/handlers.py](../../../mayan/apps/dynamic_search/handlers.py#L13-L23) + [tasks.py](../../../mayan/apps/dynamic_search/tasks.py#L41-L89) — write-path search indexing with retry/backoff.
28. [checkouts/models.py](../../../mayan/apps/checkouts/models.py#L100-L116) — check-then-act on a OneToOne; find the race.
29. [testing/tests/mixins.py](../../../mayan/apps/testing/tests/mixins.py) (outline) — descriptor-leak, tempfile-leak, random-PK test harness machinery.
30. [duplicates/duplicate_backends.py](../../../mayan/apps/duplicates/duplicate_backends.py#L8-L28) — a small, clean plugin backend to imitate when you write one.

## Drill

Pick any app you haven't read (say `web_links`). Write down, from the skeleton alone, where its permission definitions, event definitions, and API routes must be. Verify. *Basic:* 2 of 3 right. *Solid:* 3 of 3 plus you found the `apps.py` wiring. *Strong:* you can also say which reads/writes are ACL-restricted and where its tests would put a cross-tenant denial case.
