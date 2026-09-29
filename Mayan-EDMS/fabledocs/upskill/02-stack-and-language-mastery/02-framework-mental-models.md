# Framework mental models: Django + DRF + Celery

## Django ORM ≈ Prisma with opinions — plus *managers*

Model classes declare schema ([Document](../../../mayan/apps/documents/models/document_models.py#L41-L108)); migrations are generated files under each app's `migrations/`. Two ideas Prisma/TypeORM don't push on you:

**1. Managers are named query scopes with policy weight.** `Document.objects` (everything), `Document.trash` (trashed only), `Document.valid` (business-visible) — [document_models.py L106–L108](../../../mayan/apps/documents/models/document_models.py#L106-L108). Choosing a manager is an *access decision*: every API view here queries `Document.valid` ([document_api_views.py L38](../../../mayan/apps/documents/api_views/document_api_views.py#L23-L39)). The transferable idea: default-deny querysets beat per-call-site `WHERE` clauses because the safe path is the short path.

**2. Querysets are lazy, composable, and evaluate at surprising moments.** `restrict_queryset` builds Q-object trees and returns unevaluated querysets ([acls/managers.py](../../../mayan/apps/acls/managers.py#L278-L290)); truthiness/iteration executes SQL. Your React Query instinct of "the request happens when you await" maps to "the query happens when you iterate/len/bool."

**Proxy models** are a Django-specific trick used heavily here: same table, different class, different behavior/managers — `TrashedDocument`, `DocumentSearchResult` ([document_models.py L326–L336](../../../mayan/apps/documents/models/document_models.py#L326-L336)), `CheckedOutDocument` ([checkouts/models.py L119–L137](../../../mayan/apps/checkouts/models.py#L119-L137)). Why: menus, ACLs, and links dispatch on *class*, so a proxy gives one dataset several UI identities for free.

## Signals ≈ EventEmitter, with the same debts

`post_save`-style signals wire cross-app reactions: a new document file triggers OCR/parsing/search via `signal_post_document_file_upload` ([document_file_models.py L492–L500](../../../mayan/apps/documents/models/document_file_models.py#L492-L500)); search maintenance connects handlers that enqueue tasks ([dynamic_search/handlers.py](../../../mayan/apps/dynamic_search/handlers.py#L116-L125)). The repo even defines custom signals carrying a `user` kwarg ([signal_mayan_pre_save](../../../mayan/apps/documents/models/document_file_models.py#L449-L451)).

Debts (same as EventEmitter): invisible control flow (grep `apps.py` for `connect(` to find listeners), listener errors propagate into the emitter's transaction, ordering unspecified. Senior position: signals for *cross-app* decoupling, direct calls within an app — this repo mostly holds that line.

## Class-based views ≈ middleware chains built from mixins

A DRF view here is 6+ mixins deep ([rest_api/generics.py](../../../mayan/apps/rest_api/generics.py#L28-L37)). Reading order = **MRO**: `ListCreateAPIView(CheckQueryset…, DynamicFieldList…, InstanceExtraData…, …)` — each mixin overrides one hook (`get_queryset`, `perform_create`, `get_serializer_context`) and calls `super()`. Mental model: Express middleware, but composed by inheritance instead of `.use()` order.

Concrete example to study: `InstanceExtraDataAPIViewMixin.perform_create` smuggles `_event_actor` into `serializer.validated_data` ([api_view_mixins.py L153–L172](../../../mayan/apps/rest_api/api_view_mixins.py#L153-L172)) so the *model layer* can attribute the audit event to a user without changing every method signature. Clever; also the reason you'll grep for `_event_` prefixed attributes for a week. Failure mode: attribute-smuggling is invisible in signatures and breaks silently on rename — the exact bug class TS interfaces would catch.

UI views mirror the pattern with their own library ([views/mixins.py](../../../mayan/apps/views/mixins.py); `RestrictedQuerysetViewMixin` at [L549–L588](../../../mayan/apps/views/mixins.py#L549-L588)).

## DRF serializers ≈ zod schema + DTO mapper in one object

`DocumentSerializer` declares readable fields, write-only inputs (`document_type_id`), nested read shapes, and hyperlinks ([document_serializers.py L23–L62](../../../mayan/apps/documents/serializers/document_serializers.py#L23-L62)). Validation runs on `is_valid(raise_exception=True)`; DRF returns structured 400s. Two repo-specific conventions:

- `create_only_fields` (house extension) — fields accepted on POST but frozen afterwards ([Meta L44](../../../mayan/apps/documents/serializers/document_serializers.py#L43-L50)).
- Serializers may own side effects: `DocumentUploadSerializer.create` dispatches Celery work ([L74–L90](../../../mayan/apps/documents/serializers/document_serializers.py#L74-L90)). Debatable layering — great critique fodder ([03/06 architecture critique](../03-architecture-and-patterns/06-architecture-critique.md)).

## Celery ≈ BullMQ, with a decade more scar tissue

Mapping: `@app.task` ≈ queue processor; `apply_async(kwargs=...)` ≈ `queue.add`; `self.retry(exc=...)` ≈ job backoff; chord ≈ `Promise.all` + completion job (Flow 5); beat ≈ `repeat` jobs, declared per app ([documents/queues.py L50–L69](../../../mayan/apps/documents/queues.py#L50-L69)).

House rules worth stealing:
1. **IDs, not objects, cross the wire** ([documents/tasks.py L52–L72](../../../mayan/apps/documents/tasks.py#L52-L72)).
2. **`ignore_result=True` by default**; keep results only when a chord needs them ([ocr/tasks.py L51](../../../mayan/apps/ocr/tasks.py#L51-L53)). *Investigate:* one decorator says `ignore_results=True` — plural, so the option silently does nothing ([documents/tasks.py L140](../../../mayan/apps/documents/tasks.py#L140-L145)). Typo-tolerant config kwargs are a Celery sharp edge; TS would have flagged it.
3. **Retry selectively**: `OperationalError` (transient DB) retries; unknown errors surface ([sources/tasks.py L88–L93](../../../mayan/apps/sources/tasks.py#L88-L93)).

## Templates ≈ server-rendered JSX with no reactivity

UI = Django templates + Bootstrap + jQuery ([appearance/dependencies.py](../../../mayan/apps/appearance/dependencies.py#L38-L51)); interactivity is page loads and AJAX partials. State lives in the URL and the DB, not a store. When asked "where's the frontend state management?" the honest answer is "there deliberately isn't any" — and being able to defend that for a document-management UI is a better interview answer than pretending it's React.

## Interview angle

1. *"ORM lazy loading — blessing or curse?"* → show managers + the `if acl_filter:` query-executing truthiness line ([acls/managers.py L119–L123](../../../mayan/apps/acls/managers.py#L119-L123)).
2. *"How do you keep audit logging out of every controller?"* → `@method_event` + smuggled actor; discuss the tradeoff honestly.
3. *"Signals/events within a monolith: when?"* → cross-app fan-out here; name the debts.
4. *"Compare Celery and BullMQ."* → the three house rules above, plus chord semantics.
5. *"What does a serializer own?"* → validation + shape mapping; argue where the task dispatch should live.

Cards: [08/02-frontend-framework-questions.md](../08-interview-prep/02-frontend-framework-questions.md) (adapted to server-rendered UI) and [08/03](../08-interview-prep/03-api-and-data-modeling-questions.md).

**Drill:** take `APIDocumentListView` ([document_api_views.py L64–L92](../../../mayan/apps/documents/api_views/document_api_views.py#L64-L92)) and write its Express equivalent: router + zod schema + authz middleware + service call + job dispatch. Then annotate which Express lines were *implicit* in the Django version (ACL filter from the base class, event actor from the mixin, 404-on-denial from the filtered queryset). *Strong:* your annotation names all three and states where each would be forgotten first in a hand-rolled port.
