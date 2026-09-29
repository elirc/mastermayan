# Boundaries and layers

## The layer stack, with owners

| Layer | Owns | Must NOT own | Example |
| --- | --- | --- | --- |
| URLconf | routing, ID capture | logic | per-app `urls.py`, composed at startup ([common/apps.py](../../../mayan/apps/common/apps.py#L83-L89)) |
| View (UI/API) | HTTP semantics, *choosing* the permission + queryset, dispatch | business rules, SQL details | [document_api_views.py L23–L44](../../../mayan/apps/documents/api_views/document_api_views.py#L23-L44) |
| Serializer / Form | input shape, field-level validation, output shape | authz (mostly), persistence details | [document_serializers.py L23–L62](../../../mayan/apps/documents/serializers/document_serializers.py#L23-L62) |
| Model + manager | invariants, state transitions, events, queryset policy | HTTP anything, request objects | `Document.delete()` two-phase ([document_models.py L142–L163](../../../mayan/apps/documents/models/document_models.py#L142-L163)) |
| Task | orchestration across time, retries, cleanup | business rules (delegates back to models) | [documents/tasks.py L52–L124](../../../mayan/apps/documents/tasks.py#L52-L124) |
| Storage/backends | bytes, external engines | domain meaning | `DefinedStorageLazy` ([document_file_models.py L89–L92](../../../mayan/apps/documents/models/document_file_models.py#L88-L92)) |

The load-bearing rule: **models trust their callers on authorization** — `file_new` never asks who's calling ([document_models.py L189–L192](../../../mayan/apps/documents/models/document_models.py#L189-L192)); boundaries (view filters) decide access, models enforce *integrity*. This split is why the same model code serves UI, API, watch folders, and workflow actions without four authz implementations.

## Two boundaries done well

**1. The web/worker boundary.** Data crosses it only as primitive IDs plus a staged `SharedUploadedFile`; the worker re-derives everything ([sources mixins L95–L120](../../../mayan/apps/sources/source_backends/mixins.py#L95-L120) → [documents/tasks.py L64–L96](../../../mayan/apps/documents/tasks.py#L64-L96)). Contract is explicit, replayable, and survives process death. This is the pattern to cite when an interviewer asks about job queues.

**2. The engine boundary.** Search, locking, storage, MIME detection, OCR, duplicates are all `get_backend()`-resolved plugins (e.g. [LockingBackend.get_backend()](../../../mayan/apps/lock_manager/backends/base.py), [SearchBackend.get_instance()](../../../mayan/apps/dynamic_search/tasks.py#L34-L36), duplicates at [duplicate_backends.py L8–L28](../../../mayan/apps/duplicates/duplicate_backends.py#L8-L28)). Swapping Whoosh→Elasticsearch or file-lock→Redis-lock is configuration ([docker-compose.yml L12–L13](../../../docker/docker-compose.yml#L3-L16)), not surgery.

## Three leaks to learn from

**Leak 1: serializer dispatches infrastructure.** `DocumentUploadSerializer.create` creates the staging row and calls `apply_async` ([document_serializers.py L74–L90](../../../mayan/apps/documents/serializers/document_serializers.py#L74-L90)). Serializers are shape-mappers; this one now knows queue topology. Cost: can't reuse the serializer without side effects; testing needs task mocking. Fix shape: move dispatch to the view's `perform_create` or a service function. (Counterargument a maintainer might give: DRF's `create` *is* the create use-case hook; the view stays thin. Know both sides.)

**Leak 2: `_event_actor` smuggling.** The audit layer needs "who did this" deep in model methods, delivered by setting private attributes from view mixins ([api_view_mixins.py L153–L172](../../../mayan/apps/rest_api/api_view_mixins.py#L153-L172), consumed at [document_models.py L304–L316](../../../mayan/apps/documents/models/document_models.py#L304-L316)). It's a hidden parameter — a layer bypass tolerated because threading `user` through every signature would be worse. The mature take: *this is a known tax on the event pattern*, contained by naming convention (`_event_*`).

**Leak 3: model knows its cache.** `DocumentFile.cache_partition` reaches into the `file_caching` app and names its own storage key ([document_file_models.py L172–L184](../../../mayan/apps/documents/models/document_file_models.py#L172-L184)); deletion must remember to purge it ([L220–L236](../../../mayan/apps/documents/models/document_file_models.py#L220-L236)). Domain model coupled to a performance concern. Alternative: signal-driven cache invalidation. Tradeoff: explicit coupling is *greppable*; signal indirection isn't. This repo consistently chooses explicit-but-coupled — a defensible house position.

## What "boundary" buys you (say this in interviews)

A boundary is where you can: substitute implementations (engines), reason about failure independently (web vs worker), and assign blame quickly (validation error = serializer; integrity error = model; timeout = task). When someone proposes a change, the senior question is "which boundary does this cross, and does the contract change?" — e.g. adding a field to an API response crosses only the serializer (cheap); renaming a task kwarg crosses the queue contract (deploy-order hazard, see [02/03 case study 3](../02-stack-and-language-mastery/03-type-system-and-contracts.md)).

**Drill:** classify these five changes by boundary crossed and risk: (a) add `description` max length validation; (b) rename `task_document_upload` → `task_upload_document`; (c) switch document storage to S3; (d) make trash-emptying respect per-type order; (e) add a `?fields=` param to document list. Check yourself against the layer table. *Strong:* for (b) you describe the two-phase deploy (new name added, old kept as alias, then removed) and for (c) you note it's config-only *except* for anything assuming local filesystem semantics.

**Interview angle:** "Describe the architecture of a codebase you know" — use the layer table verbatim; then volunteer Leak 1 with both sides. Volunteering a critique *with the counterargument* is the strongest mid-level signal there is. Cards: [08/04](../08-interview-prep/04-system-design-from-this-repo.md).
