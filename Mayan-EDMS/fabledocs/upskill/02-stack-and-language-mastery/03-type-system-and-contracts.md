# Type system and contracts: living without TypeScript

Python 3 here is essentially **untyped** — this 2022 codebase has no annotations, no mypy. So where TS would enforce contracts at compile time, this repo enforces them at *specific runtime checkpoints*. Learning to see those checkpoints is the transferable skill: every codebase, typed or not, has a finite set of places where bad data is stopped.

## The contract checkpoints, ranked by strength

| Checkpoint | Enforced when | Example anchor | TS analog |
| --- | --- | --- | --- |
| Database schema | INSERT/UPDATE | `checksum = CharField(max_length=64, db_index=True)` ([document_file_models.py](../../../mayan/apps/documents/models/document_file_models.py#L110-L121)); `unique_together` on cache files ([file_caching/models.py](../../../mayan/apps/file_caching/models.py#L347-L351)); `OneToOneField` on checkout ([checkouts/models.py](../../../mayan/apps/checkouts/models.py#L32-L35)) | none — DB constraints outrank types |
| DRF serializer fields | request parse | `PrimaryKeyRelatedField(queryset=...)` resolves an int **into an instance** and 400s on garbage ([document_serializers.py](../../../mayan/apps/documents/serializers/document_serializers.py#L65-L68)) | zod schema |
| Model `clean()` | form/serializer full_clean | checkout expiration must be future ([checkouts/models.py](../../../mayan/apps/checkouts/models.py#L68-L72)); workflow log entry validates transition ([workflow_instance_models.py](../../../mayan/apps/document_states/models/workflow_instance_models.py#L249-L251)) | refinement functions |
| Registry validation | startup / registration | `ModelPermission` rejects permissions not registered for a class ([acls/managers.py](../../../mayan/apps/acls/managers.py#L359-L362)) | exhaustive unions |
| Duck typing + conventions | never (crashes at use) | `_event_actor` smuggling; task kwargs | the stuff TS exists for |

**Interview vocabulary:** the DB layer enforces **invariants** (always true), serializers enforce **input contracts** (true at the boundary), and everything in the last row is **convention** — cheap until it isn't.

## Case study 1: a name that lies, safely

`DocumentChangeTypeSerializer.document_type_id` is *named* like an int but is a `PrimaryKeyRelatedField` — `validated_data['document_type_id']` is a **DocumentType instance** ([document_serializers.py L65–L68](../../../mayan/apps/documents/serializers/document_serializers.py#L65-L68)), which is why passing it straight into `document_type_change(document_type=...)` works ([document_api_views.py L106–L110](../../../mayan/apps/documents/api_views/document_api_views.py#L106-L110)). In TS the type would contradict the name and someone would fix it. Here the name misleads every new reader — a real review comment waiting to happen (and Ticket 8 in [06-contribution](../06-contribution-practice/01-good-first-tickets.md)).

## Case study 2: convention breaks silently

Celery accepts unknown decorator kwargs; `ignore_results=True` (plural) type-checks nowhere and does nothing ([documents/tasks.py L140](../../../mayan/apps/documents/tasks.py#L140-L145)). Compare the correct `ignore_result=True` two screens up ([L129](../../../mayan/apps/documents/tasks.py#L129-L137)). This is the exact class of bug (`strict` object literals) TypeScript eliminated for you. Without types, the defenses are: linters, tests that assert behavior (is the result actually discarded?), and reviewers who know the API surface.

## Case study 3: contracts across the queue

Task signatures are inter-process contracts: `task_document_file_upload(document_id, shared_uploaded_file_id, user_id=None, action=None, comment=None, expand=False, filename=None)` ([documents/tasks.py L52–L55](../../../mayan/apps/documents/tasks.py#L52-L55)). Producers build these kwargs as dict literals in another app ([sources mixins L82–L91](../../../mayan/apps/sources/source_backends/mixins.py#L82-L93)). Rename a kwarg and old queued messages — or un-updated producers — fail at *delivery time* in a worker, far from the edit. Mitigations visible here: kwargs-only style (no positional drift), additive-only evolution, defaults on everything new. That's protobuf discipline without protobuf.

## Case study 4: `natural_key` as identity contract

Models define `natural_key()` (Document → its UUID, [document_models.py L276–L278](../../../mayan/apps/documents/models/document_models.py#L276-L278); DocumentFile → checksum + document key, [document_file_models.py L363–L365](../../../mayan/apps/documents/models/document_file_models.py#L363-L365)) so fixtures/export can reference rows stably across databases where auto-increment PKs differ. Transferable: **identity is a design decision** — surrogate key for FK graphs, natural key for portability, UUID for merge-ability. Interviewers love this axis.

## How to *add* safety here without a rewrite

1. Tests as types: the `no_permission`/`with_access` pairs ([test_document_api.py L27–L66](../../../mayan/apps/documents/tests/test_document_api.py#L27-L66)) pin the authz contract per endpoint.
2. Constants modules (`literals.py` in every app) instead of magic strings.
3. Fail-fast registries: `ModelPermission.register` makes invalid grants raise at grant time, not read time ([acls/managers.py L359–L362](../../../mayan/apps/acls/managers.py#L359-L362)).
4. If you were adding annotations today: start at task boundaries and serializer `create/update` — highest cross-boundary risk per line.

## Interview angle

1. *"You're used to TS — how do you stay safe in a dynamic codebase?"* → name the checkpoint table; give the `ignore_results` story as the failure and the DB constraint as the backstop.
2. *"unknown vs any"* (pure TS question) → map to Python: `object` you must inspect vs "whatever, crash later"; this repo's `kwargs.get(...)` chains are `any`-culture, its serializer fields are `unknown`-culture.
3. *"Where should validation live?"* → boundary (serializer) for shape, model `clean()` for business rules, DB for invariants — with the checkout expiration example spanning all three.

Cards: [08/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q8–Q11.

**Drill:** find three places `DocumentFile.save()` would accept nonsense that the DB later rejects, and one place nonsense sails all the way to storage. *Basic:* max_length overflows. *Solid:* null mimetype path (`mimetype_update` swallows all exceptions and writes empty strings, [document_file_models.py L344–L361](../../../mayan/apps/documents/models/document_file_models.py#L344-L361)). *Strong:* you argue whether that swallow is a feature (unknown formats are legal) or a data-quality leak, and design the log line that would tell you which.
