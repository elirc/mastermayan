# Senior Build Projects

## Project 1: Make upload pipeline more observable
Problem statement: Async `202` upload acceptance hides downstream failure modes.
Product value: Faster support and safer operations.
Design checklist: correlation ids, temp upload lifecycle, callback visibility, privacy-safe logs.
Likely files/modules: [`sources/tasks.py`](../../mayan/apps/sources/tasks.py), [`sources/models.py`](../../mayan/apps/sources/models.py)
Migration plan: no-schema logging first, metrics second.
Test plan: upload success/failure integration tests.
Security plan: no file contents in logs.
Performance plan: ensure logging overhead is modest.
Rollout/rollback plan: feature-flag extra instrumentation.
Open questions: what is the canonical upload correlation key?

## Project 2: Reduce hidden coupling in document-file post-save work
Problem statement: `DocumentFile.save()` coordinates persistence, derived fields, and follow-on behavior.
Product value: lower latency, clearer ownership, easier debugging.
Design checklist: ordering, idempotency, backward compatibility, visibility.
Likely files/modules: [`document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py), related tasks and signals.
Migration plan: instrument first, then move one responsibility at a time.
Test plan: upload regression suite, failure-path tests, derived-field correctness.
Security plan: preserve audit/user context.
Performance plan: compare request latency before and after.
Rollout/rollback plan: feature-gate any async extraction.
Open questions: which derived fields must be available immediately?

## Project 3: Build a maintainers' signal map and targeted refactors
Problem statement: signal-driven coupling is powerful but easy to miss during review.
Product value: safer changes and easier onboarding.
Design checklist: trigger, target, side effect, ownership, failure visibility.
Likely files/modules: app configs, handlers, signal registrations.
Migration plan: document first, refactor second.
Test plan: verify behavior around any extracted signal path.
Security plan: ensure auth-sensitive flows keep the same boundaries.
Performance plan: identify task storms before refactoring.
Rollout/rollback plan: one chain at a time.
Open questions: which signal chains are most incident-prone?

## Project 4: Introduce robust temp-upload cleanup and reporting
Problem statement: temp uploads are a critical reliability seam.
Product value: better storage hygiene and debugging.
Design checklist: active-vs-orphan distinction, retention, operator UX.
Likely files/modules: [`sources/tasks.py`](../../mayan/apps/sources/tasks.py), storage models, management commands.
Migration plan: reporting first, cleanup automation second.
Test plan: failure-path tests and orphan-detection tests.
Security plan: avoid leaking file names to unauthorized viewers.
Performance plan: sweeper must scale with temp-upload volume.
Rollout/rollback plan: dry-run mode before deletion mode.
Open questions: what retention window is acceptable?

## Project 5: Improve ACL reasoning ergonomics for contributors
Problem statement: secure patterns exist, but they are easy to bypass accidentally.
Product value: fewer authorization regressions.
Design checklist: helper design, discoverability, testability, review clarity.
Likely files/modules: API views, helper modules, docs, tests.
Migration plan: add helper and docs, migrate one flow at a time.
Test plan: permission regression tests for migrated flows.
Security plan: preserve current least-privilege semantics exactly.
Performance plan: helper must not hide expensive query behavior.
Rollout/rollback plan: incremental adoption.
Open questions: which repeated permission patterns are stable enough to abstract?
