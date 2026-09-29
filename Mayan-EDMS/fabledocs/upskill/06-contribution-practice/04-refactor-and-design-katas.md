# Refactor and design katas

Senior katas. No merge required — the deliverable is a written design + self-grade. Each names the trap and a rubric.

## Kata 1: Extract the task-dispatch side effect out of the serializer

**Trap:** `DocumentUploadSerializer.create` owns queue knowledge ([document_serializers.py L74–L90](../../../mayan/apps/documents/serializers/document_serializers.py#L74-L90) — [leak 1](../03-architecture-and-patterns/01-boundaries-and-layers.md)).
**Task:** design the move to a service function or `perform_create`, keeping the API contract identical.
**Self-grade:** *Solid* — dispatch moves, tests still green. *Strong* — you show the counterargument (DRF's `create` *is* the use-case hook) and pick a side with reasons, plus note every caller of the serializer.

## Kata 2: Design the workflow transition concurrency fix

**Trap:** no lock; last-write-wins on derived state (Flow 6, [workflow_instance_models.py L82–L104](../../../mayan/apps/document_states/models/workflow_instance_models.py#L82-L104)).
**Task:** compare `select_for_update` vs expected-predecessor + unique constraint; write the migration and the failing-then-passing test.
**Self-grade:** *Strong* — you weigh lock-hold time against the inline state actions ([L282–L311](../../../mayan/apps/document_states/models/workflow_instance_models.py#L282-L311)) and argue the constraint approach avoids holding a lock across side effects.

## Kata 3: Introduce an outbox for event fan-out

**Trap:** synchronous notification creation inside `save` ([events/classes.py L387–L427](../../../mayan/apps/events/classes.py#L359-L429)).
**Task:** design a transactional outbox: write the audit action synchronously, enqueue notification creation. Specify the ordering guarantee change and the dedupe key.
**Self-grade:** *Strong* — your design keeps audit atomic with the mutation, makes notifications eventual, and states the user-visible change ("notification may arrive after reload").

## Kata 4: Split the `documents` app's `models/` responsibilities

**Trap:** `DocumentFile.save` does hooks + signals + events + derived fields + parent mutation ([document_file_models.py L428–L504](../../../mayan/apps/documents/models/document_file_models.py#L428-L504)) — one method, five concerns.
**Task:** propose a decomposition (a `DocumentFilePipeline` service?) that preserves the transaction boundary and every event.
**Self-grade:** *Strong* — you identify which steps *must* stay in one transaction ([L466–L484](../../../mayan/apps/documents/models/document_file_models.py#L466-L484)) and refuse to split those, showing you optimize for the invariant, not for line count.

## Kata 5: Remove the `_event_actor` smuggling (or decide not to)

**Trap:** hidden parameter via instance `__dict__` ([api_view_mixins.py L153–L172](../../../mayan/apps/rest_api/api_view_mixins.py#L153-L172), [events/classes.py L161–L184](../../../mayan/apps/events/classes.py#L161-L184)).
**Task:** design an explicit-actor alternative (thread `user` through, or an `EventContext` object) and estimate the diff size across the repo.
**Self-grade:** *Strong* — you conclude, with evidence, that the cure (400 signature changes) is worse than the disease, and instead propose *containing* it (a typed helper + docs). Knowing when **not** to refactor is the kata's real lesson.

## Kata 6: Improve type safety at one boundary without a rewrite

**Trap:** untyped task kwargs rot silently ([documents/tasks.py L140](../../../mayan/apps/documents/tasks.py#L140-L145)).
**Task:** add annotations + a permissive mypy gate to `documents/tasks.py`; show the config and the first bug it would catch.
**Self-grade:** *Strong* — incremental (one file), CI-optional, and you name the social risk (does the project want typing?) as the primary blocker.

## Kata 7: Design a migration that changes `active` version enforcement to a DB constraint

**Trap:** "≤1 active version per document" is enforced by transactional convention, not a constraint ([document_version_models.py L95–L101, L373–L378](../../../mayan/apps/documents/models/document_version_models.py#L95-L101)).
**Task:** design a partial unique index (`WHERE active`) migration, including backfill for any existing violations and the reverse migration.
**Self-grade:** *Strong* — you detect and repair pre-existing violations in the data migration *before* adding the constraint (or the migration fails on real data), and provide a reverse function.

## Kata 8: Write the RFC for the ACL materialization spike

**Trap:** the biggest, riskiest change (Project 2 option c).
**Task:** produce a one-page RFC (template in [07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)): problem with evidence, two options, recommendation, migration, test plan, rollback, open questions.
**Self-grade:** *Strong* — your RFC leads with "measure first," proposes the cheap wins before the expensive bet, and honestly lists what could make it not worth doing.

---

**Meta-rubric for all katas.** A junior refactors to make code *prettier*. A mid-level refactors to reduce a *specific* future cost and can name it. A senior sometimes writes "no change — here's why the current shape is correct for this system," and that answer scores full marks (katas 5 and 8 reward it). Every kata: state the invariant you must not break *before* you propose the change.
