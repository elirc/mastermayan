# Mid-level feature tickets

Twelve cross-layer tickets. Each requires a **design note before code** (problem, approach, alternatives, risk, rollback) — that note is the mid-level skill being trained. Each touches ≥2 layers (schema/API/UI/task/test).

---

## Ticket M1: User-visible feedback for version modifications

**Difficulty:** Medium — **1–2 days.** **Skills:** signals, messaging, async UX.
Currently "append/reset pages" are fire-and-forget with no success/failure signal ([document_version_modifications.py](../../../mayan/apps/documents/document_version_modifications.py#L10-L40), tasks at [documents/tasks.py L219–L251](../../../mayan/apps/documents/tasks.py#L218-L251)). Add a `messaging` notification on completion/failure, following the export pattern that already messages users ([document_version_models.py L164–L193](../../../mayan/apps/documents/models/document_version_models.py#L145-L193)).
**Design note must cover:** which app owns the notification, ordering (task success vs message), and whether to also fix the suspect `order_by` ([02-data-model drill](../03-architecture-and-patterns/02-data-model-and-persistence.md)) as part of "completion" (arguably you can't claim success while it's broken).
**Risk/rollback:** additive; behind nothing; revert = remove the message call.
**Interview story potential:** "closing a silent-failure gap in an async feature."

## Ticket M2: Restrict change-type target by `document_create` (behind a setting)

**Medium–Hard — 2–3 days.** Close the authz asymmetry ([03/03 §edge cases](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)) behind a default-off setting, covering **both** API ([document_serializers.py L65–L68](../../../mayan/apps/documents/serializers/document_serializers.py#L65-L68)) and UI ([document_views.py L80–L90](../../../mayan/apps/documents/views/document_views.py#L80-L90)) paths.
**Design note:** the serializer needs the request user in context (static queryset → method); the existing test enshrines the loose behavior so this needs a changelog + release note; deprecation flip plan.
**Risk/rollback:** setting-gated → reversible; the review kata 6 ([04/04](../04-code-reading-gym/04-review-katas.md)) is the pre-mortem.
**Story potential:** "a security fix that was 20% code, 80% rollout process."

## Ticket M3: Dead-letter table for exhausted tasks

**Hard — 3 days.** A `FailedTask` model + a shared retry-exhaustion handler that writes to it, wired into the search/upload tasks first ([dynamic_search/tasks.py L60–L87](../../../mayan/apps/dynamic_search/tasks.py#L41-L89)). Admin list view to inspect/retry.
**Design note:** Celery signal (`task_failure`) vs explicit calls; PII in stored kwargs; retention.
**Risk:** new model = migration; keep it additive and opt-in per task.
**Story potential:** the observability story ([08/06](../08-interview-prep/06-behavioral-star-stories.md)).

## Ticket M4: `assertNumQueries` regression harness for the 5 hottest list views

**Medium — 2 days.** Systematize [04/perf](../05-quality-engineering/04-performance-thinking.md): pick 5 list endpoints, add query-count tests at N=1 and N=10, fix any that scale with N (likely `file_latest`/`version_active` per-row properties, [document_models.py L185–L187](../../../mayan/apps/documents/models/document_models.py#L185-L187)).
**Design note:** where to inject `Prefetch`/annotation (view `get_source_queryset` vs `ModelQueryFields`).
**Story potential:** "turned performance folklore into CI gates."

## Ticket M5: Workflow transition concurrency guard

**Hard — 3 days.** Prevent the double-transition race (Flow 6). Implement expected-predecessor: log entry stores the state it transitions *from*; a unique/`select_for_update` guard rejects the loser.
**Design note:** DB constraint vs row lock (contention vs hold time), and the impact on inline state actions ([workflow_instance_models.py L282–L311](../../../mayan/apps/document_states/models/workflow_instance_models.py#L282-L311)).
**Story potential:** distributed-state correctness ([08/04 variation 2](../08-interview-prep/04-system-design-from-this-repo.md)).

## Ticket M6: Async event fan-out for notifications

**Hard — 3–4 days.** Move the synchronous subscriber/notification creation out of `EventType.commit` into a queued task ([events/classes.py L387–L427](../../../mayan/apps/events/classes.py#L359-L429)); keep the audit `action.send` synchronous.
**Design note:** ordering guarantee change (notification eventual); at-least-once dedupe; the blast radius (every save in the app touches this path).
**Risk:** high — this is the critique's "one structural bet" ([03/06](../03-architecture-and-patterns/06-architecture-critique.md)); RFC required.
**Story potential:** decoupling a hot path.

## Ticket M7: Rate-limit the token-obtain endpoint

**Medium — 1–2 days.** Add DRF throttling to the auth token endpoint ([rest_api/urls.py L12–L16](../../../mayan/apps/rest_api/urls.py#L12-L16)) and document the setting.
**Design note:** throttle scope (per-IP vs per-user), storage (cache backend), and interaction with the browsable API.
**Story potential:** "closed a brute-force surface."

## Ticket M8: Watch-folder batch ingestion with lock-budget awareness

**Medium–Hard — 2–3 days.** Implement kata 8's safe version ([04/04](../04-code-reading-gym/04-review-katas.md)): ingest up to N files or T seconds per tick, respecting `DEFAULT_SOURCES_LOCK_EXPIRE` ([sources/tasks.py L24–L33](../../../mayan/apps/sources/tasks.py#L18-L55)).
**Design note:** batch bound derivation from lock timeout; fairness across sources; memory.
**Story potential:** throughput-vs-safety tradeoff under a distributed lock.

## Ticket M9: Cache prune efficiency + hit decay

**Hard — 3 days.** The prune loop recomputes a SUM per iteration ([file_caching/models.py L114–L152](../../../mayan/apps/file_caching/models.py#L105-L152)) and `hits` never decays (old-hot files immortal). Batch the size computation; add time-decay to eviction ordering.
**Design note:** measure current cost first; eviction policy change is behavioral (which files survive).
**Story potential:** cache eviction design.

## Ticket M10: Storage/DB reconciliation report

**Medium–Hard — 2–3 days.** A management command / periodic task comparing `DocumentFile` rows to storage objects, reporting orphans (the class-(c) residue from [04-side-effects drill](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).
**Design note:** read-only report vs auto-cleanup (never auto-delete storage on first version!); pagination for large stores.
**Story potential:** data-integrity tooling.

## Ticket M11: Effective-permissions inspector

**Medium–Hard — 2–3 days.** A view/command that, given a user + object, explains *why* access is (not) granted, walking `get_inherited_permissions` ([acls/managers.py L296–L310](../../../mayan/apps/acls/managers.py#L296-L310)).
**Design note:** where to surface (admin vs API); performance (this is the expensive recursion).
**Story potential:** "made an opaque authz system debuggable."

## Ticket M12: Archive expand depth/size limits

**Medium — 2 days.** Implement ticket 19's design: max-depth + total-uncompressed-size limits on `file_new(expand=True)` ([document_models.py L205–L224](../../../mayan/apps/documents/models/document_models.py#L189-L251)), with a setting and clear errors.
**Design note:** where to enforce (before or during expansion), and the UX when a limit is hit mid-archive.
**Story potential:** zip-bomb DoS mitigation.

---

**Rubric for your design note (self-grade):** *Basic* — states the change and the files. *Solid* — includes alternatives considered and a rollback. *Strong* — quantifies blast radius, names the test that proves it, and identifies the one thing that would make a maintainer say no *before* they say it. The [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) shows the target voice.
