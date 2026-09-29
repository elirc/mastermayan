# Debugging and code-review rounds — timed simulations

Practical rounds test how you *think out loud* under mild pressure. Set a timer; narrate everything; the interviewer wants your process, not a fast answer. Grading rubric at the end applies to all.

---

## Debugging round 1 (15 min): "Uploaded document never appears"

**Setup given to you:** "A user uploads via the API, gets a 201, but the document never shows in their list. Walk me through debugging this."
**What they're testing:** async reasoning, hypothesis ordering, knowing when to look server-side vs client-side.
**The path to narrate** (from [05/03 scenario 1](../05-quality-engineering/03-systematic-debugging.md)): "First I'd separate the async halves — the 201 means the row exists but the file task may not have run. I'd check `is_stub` in the DB; if true, the worker failed or isn't running — check the uploads queue and worker logs. If `is_stub` is false, it's a *visibility* problem: ACL (does the user have view via the type?) or the wrong manager. I'd reproduce the exact authz decision in a shell with `restrict_queryset(...).filter(pk=X).exists()`."
**Interviewer follow-ups they'll throw:** "worker logs are clean, is_stub is false" → pivot to ACL. "The user is an admin" → staff bypass should grant it, so suspect the manager or an exception in `get_context_data` ([document_views.py L40–L54](../../../mayan/apps/documents/views/document_views.py#L36-L55)). "It works in staging not prod" → config (lock backend host-local? different worker deployment?).
**Strong signal:** you state the branching hypothesis *before* touching anything and you name the cheapest probe at each fork.

## Debugging round 2 (15 min): "This background action silently does nothing"

**Setup:** the version "append all pages" modification ([05/03 scenario 2](../05-quality-engineering/03-systematic-debugging.md)).
**Narrate:** "Fire-and-forget task, so no user error surfaces — first I confirm the task dispatched (worker_b logs), then I look for a traceback. If I see a `FieldError` on `document_file__page_number`, that field lives on `DocumentFilePage`, not `DocumentFile` ([document_file_page_models.py L38](../../../mayan/apps/documents/models/document_file_page_models.py#L38)) — the `order_by` is wrong ([document_version_models.py L299–L301](../../../mayan/apps/documents/models/document_version_models.py#L291-L309)). Cheapest confirmation: call `pages_append_all()` in a shell, no Celery needed."
**Follow-ups:** "how would you prevent this class of bug?" → a model-level test invoking the method directly + surfacing task failures to the user. "It only fails for some documents" → data-dependent; multi-file documents hit the ordering path.
**Strong signal:** you treat the *silent* part as a bug equal to the ordering part.

## Debugging round 3 (15 min): "Intermittent 500s on page images under load"

**Setup:** [05/03 scenario 4](../05-quality-engineering/03-systematic-debugging.md).
**Narrate:** "`LockError` under load points at cache-partition lock contention. I'd correlate failures with prune activity — a too-small cache forces a prune on every create, holding locks longer. So I check cache utilization vs `maximum_size` and lock backend (file-lock is host-local — wrong for multi-host). The fix is likely capacity + config, not code."
**Follow-up:** "the cache is huge and it still happens" → then it's genuine lock contention on hot partitions; batch or shorten the critical section.
**Strong signal:** you consider "correct code + wrong config" as a root cause, not just code bugs.

## Debugging round 4 (10 min): "Two users both approved the same request"

**Setup:** workflow race ([05/03 scenario 5](../05-quality-engineering/03-systematic-debugging.md)).
**Narrate:** "State is derived from the last log entry with no lock ([workflow_instance_models.py L82–L104](../../../mayan/apps/document_states/models/workflow_instance_models.py#L82-L104)), so two transitions validate against the same current state and both append. I can reproduce without threads — two sequential `do_transition` calls without re-reading state. Fix options: `select_for_update` (holds a lock across inline state actions — bad) or expected-predecessor + unique constraint (DB-enforced, no lock hold)."
**Strong signal:** you model the race as sequential calls and weigh the two fixes on lock-hold-time.

---

## Code-review round 1 (15 min): review kata 4 — the `is_stub` "fix"

Present the diff ([04/04 kata 4](../04-code-reading-gym/04-review-katas.md)). **What they want:** do you rubber-stamp an "obvious" fix or investigate? The winning move: "This one-liner is load-bearing — flipping `is_stub` to True re-exposes the document to the stub reaper, which would auto-delete documents whose last file was removed. I can't approve either behavior without a test capturing intent." **Strong signal:** you refuse both directions until a decision exists, and your comment proposes the deciding test.

## Code-review round 2 (15 min): review kata 2 — retry-on-any-exception

Present the OCR retry-broadening diff ([04/04 kata 2](../04-code-reading-gym/04-review-katas.md)). **Want:** you catch that corrupt-file failures now retry forever, burning the OCR worker and stalling the chord finisher for healthy pages. **Strong signal:** you ask for the observed error class before approving *any* retry change, and you point at the existing named-exception taxonomy as precedent.

## Code-review round 3 (15 min): review kata 6 — restrict change-type target

Present the authz-tightening diff ([04/04 kata 6](../04-code-reading-gym/04-review-katas.md)). **Want:** you recognize this changes a *test-enshrined* contract (process issue), and that the serializer needs request context to know the user (technical issue), and that an API-only fix ignores the UI path. **Strong signal:** you separate "must fix in this PR" from "must socialize/changelog," showing you see process as part of engineering.

---

## Universal grading rubric

| Level | Debugging | Review |
| --- | --- | --- |
| Basic | jumps to a fix; guesses | finds the surface issue |
| Solid | narrates hypotheses, picks cheap probes, localizes layer | finds the blocking issue + a test gap |
| Strong | states branch structure first; distinguishes not-yet/denied/destroyed and code/config; adds the regression test | refuses to approve ambiguous behavior; separates code from process; quantifies impact; cites in-repo precedent |

**Meta-drill:** do rounds 1, 2, and review 1 with a 15-minute timer and a recording. Listen back for: did you say what you were checking *before* checking? Did you name a probe's cost? Did you end with a regression test? Those three habits are the difference between "smart" and "hireable at mid-level."
