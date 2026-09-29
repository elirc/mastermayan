# Systematic debugging

The method: **reproduce → narrow the layer → hypothesize → test the cheapest hypothesis first → fix the root cause → add regression coverage.** The repo-specific twist: half of all symptoms here have an async cause, so step one of narrowing is always *"did this happen in the web process or a worker?"* Tools you actually have: Django `runserver` + debugger, `./manage.py shell` for queryset probes, worker logs (per-queue), the per-entity error logs in the UI ([pattern 12](../03-architecture-and-patterns/05-pattern-catalog.md)), `QuerySet.query` for SQL shape, and the events list as a poor man's trace.

Each scenario below: try it on paper before reading the path.

## Scenario 1: "I uploaded a document and it's not in the list"

Reproduction: upload via API, immediately GET `/documents/`.
First question: web or worker? (Upload is split — Flow 1.)
Narrowing path:
1. Does the API response contain the document ID? Yes → row exists.
2. `Document.objects.filter(pk=X).values('is_stub','in_trash')` in shell → `is_stub=True`? The file task hasn't run/failed.
3. Check `uploads` queue depth and worker_c logs; check `SharedUploadedFile` count (orphans = task died after staging).
4. If `is_stub=False` but invisible → it's the *list's* filters: ACL (does the user have `document_view` via type?) or manager (list uses `Document.valid`).
Useful probes: `restrict_queryset(...).filter(pk=X).exists()` in shell reproduces the exact authz decision ([acls/managers.py L268–L294](../../../mayan/apps/acls/managers.py#L268-L294)).
Likely root causes, in base-rate order: worker not running; ACL missing; stub reaped (waited too long, [documents/tasks.py L129–L137](../../../mayan/apps/documents/tasks.py#L129-L137)).
Regression test: recipe 5 in [02-writing-tests-here](02-writing-tests-here.md).
Senior lesson: in async systems, "not there" has ≥3 distinct meanings (not yet / denied / destroyed) — instrument to distinguish them.
Interview narration: state the split-brain hypothesis out loud first ("the API path is async; I'd first establish which half failed") — that sentence alone signals mid-level.

## Scenario 2: "Version modification 'append all pages' does nothing"

Reproduction: run the modification from the UI; page list unchanged; no error shown.
First question: was the task even dispatched? (fire-and-forget, [document_version_modifications.py L19–L26](../../../mayan/apps/documents/document_version_modifications.py#L10-L26)).
Narrowing path: worker_b logs for `task_document_version_page_list_append` → expect a traceback; if `FieldError` on `document_file__page_number`, you've found the suspect ordering ([document_version_models.py L299–L301](../../../mayan/apps/documents/models/document_version_models.py#L291-L309), *investigate* per [02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
Cheapest hypothesis test: `./manage.py shell` → call `document_version.pages_append_all()` directly — 30 seconds, no Celery.
Fix root cause, not symptom: correct the ordering path *and* surface task failures to the user (messaging), else the next silent failure waits.
Regression: model-level test invoking `pages_append_all` with 2 files.
Senior lesson: **silent async failure is a product bug and a code bug**; fix both or you'll be back.

## Scenario 3: "Search finds the document by label but not by its tag"

First question: is this an indexing lag or a mapping gap?
Narrowing: 1) re-save the document; wait; retry (lag test). 2) If still missing, check the related-path registration for tags → documents in search declarations and the m2m handler wiring ([dynamic_search/handlers.py L84–L113](../../../mayan/apps/dynamic_search/handlers.py#L84-L113)). 3) Worker logs for `task_index_related_instance_m2m` errors.
Probes: run the backend query directly via `SearchBackend.get_instance().search(...)` in shell; inspect the Whoosh/ES doc for the entity.
Likely causes: task retries exhausted (LockError storms), or the reverse-path resolution returned empty ([handlers L60–L64](../../../mayan/apps/dynamic_search/handlers.py#L56-L81)).
Regression: search test that adds a tag then indexes eagerly.
Senior lesson: read-model bugs need *two* localizations: write-side (did the update dispatch?) and read-side (did the query look where the update wrote?).

## Scenario 4: "Every page image intermittently 500s under load"

Symptom: `LockError` in logs; page images sometimes render, sometimes error.
First question: contention on what lock? Cache-partition file locks ([file_caching/models.py L406–L435](../../../mayan/apps/file_caching/models.py#L406-L435)).
Narrowing: correlate failures with cache prune activity — a too-small `maximum_size` forces prune on every create ([L114–L152](../../../mayan/apps/file_caching/models.py#L105-L152)), holding locks longer; combined with `FileCachingException('Too many cache prunes…')` you have a sizing problem, not a code problem.
Probes: cache utilization via `Cache.get_total_size()` vs `maximum_size`; hit rates on `CachePartitionFile.hits`.
Fix: resize cache / verify lock backend is Redis in multi-host deploys (file lock is host-local! [04-side-effects §locks](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).
Senior lesson: some bugs are **configuration interacting with correct code** — the fix is capacity math plus a better error message, and the regression test is a monitoring alert.

## Scenario 5: "Two users approved the same invoice; workflow shows both transitions"

Reproduction: two sessions POST the same transition near-simultaneously.
Narrowing: read the log entries — both validated against the same origin state (no lock, [workflow_instance_models.py L82–L104](../../../mayan/apps/document_states/models/workflow_instance_models.py#L82-L104)); state derived from *last* entry so the second write silently "wins."
Cheapest confirmation: model test with two `do_transition` calls without refetching state (no threads needed — the check is the race).
Fix options (argue tradeoffs, Flow 6 drill): `select_for_update` on the instance row; or expected-predecessor column + unique constraint (DB-enforced, no lock hold).
Regression: test asserting the second conflicting transition raises/no-ops.
Senior lesson: races rarely need real concurrency to test — **model the interleaving as sequential calls that skip the re-read.**

**Interview versions** of scenarios 1, 2, and 5 with timers and follow-up scripts: [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).

**Drill:** for each scenario write the *one log line or metric* that would have led you to the root cause in under 5 minutes. That list is your observability backlog — compare with [06-observability](06-observability-and-operations.md).
