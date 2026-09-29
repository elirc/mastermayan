# Annotation drills

Method: open the excerpt, and **without running anything**, write five annotations: (1) inputs & their trust level, (2) outputs/return contract, (3) invariants assumed or enforced, (4) side effects, (5) failure modes. Then check against the notes. Grade: *Basic* = 3 categories mostly right; *Solid* = all 5 with at least one non-obvious item; *Strong* = you found something the notes missed (it happens — file an issue for yourself in [06/01](../06-contribution-practice/01-good-first-tickets.md) style).

## Drill 1: `Document.file_new` — [document_models.py L189–L251](../../../mayan/apps/documents/models/document_models.py#L189-L251)

Notes: inputs include an open `file_object` whose *position* matters, and `expand` triggering **recursion** for archives (L205–L224); trust level "internal callers only" — no authz. Output: `DocumentFile` or `None` (the expand path returns nothing! L224 — callers wanting the file can't use expand). Invariants: default action resolves to `DocumentFileActionUseNewPages` (L195–L196). Side effects: file rows, storage writes, action execution (page remapping), logging. Failures: archive bombs recurse unboundedly (nested `expand=True` comment at L210–L213 admits office-file risk); exceptions logged then re-raised (L237–L242).

## Drill 2: `AccessControlList.restrict_queryset` — [acls/managers.py L268–L294](../../../mayan/apps/acls/managers.py#L268-L294)

Notes: anonymous short-circuit (none), direct-role short-circuit (all — including staff/superuser via `user_has_this`), else Q-filter OR-reduction. Non-obvious: `PermissionDenied` is used as **control flow** (the `except` branch is the *normal* path for ACL users). Failure mode: `final_query` starts as `None`; if `_get_acl_filters` ever returned an empty list, `queryset.filter(None)` would raise — invariant: case 1 always appends at least one Q (L156–L166).

## Drill 3: `CachePartition.create_file` — [file_caching/models.py L225–L272](../../../mayan/apps/file_caching/models.py#L225-L272)

Notes: it's a `@contextmanager` — the caller's `with` body runs at the `yield` (L249). Side effects *before* yield: prune (may delete other files!), delete-then-create empty storage object (L237–L243). Error path: rolls back row or storage object depending on how far it got (L250–L264). Invariant: everything under a named distributed lock; nested helpers take `_acquire_lock=False` to avoid self-deadlock — an idiom worth memorizing. Failure: `LockError` propagates to caller (L270–L272) — callers must treat "cache busy" as retryable.

## Drill 4: `EventManager.pop_event_attributes` — [events/classes.py L161–L184](../../../mayan/apps/events/classes.py#L161-L184)

Notes: reads-and-*removes* `_event_*` attributes from `instance.__dict__` (pop!), so attributes are one-shot per save unless listed in `keep_attributes`. Called **twice** per decorated method (before and after the wrapped call — [decorators.py L17–L26](../../../mayan/apps/events/decorators.py#L8-L33)) so values set *inside* the method also count. Failure mode: a second save on the same instance silently loses the actor → events with `actor=None`. This explains dozens of `self._event_actor = user` re-assignments across the repo.

## Drill 5: `DocumentCheckout.save` — [checkouts/models.py L100–L116](../../../mayan/apps/checkouts/models.py#L100-L116)

Notes: rejects updates entirely (`not is_new` → raise — checkouts are immutable leases). Check-then-act race: `is_checked_out()` then `save()`; DB `OneToOneField` (L32–L35) backstops with `IntegrityError`, so the *exception type* differs by race timing — callers catching only `DocumentAlreadyCheckedOut` miss the race loser. Side effect: event commit *after* successful insert (L106–L109). Compare with `clean()` (L68–L72): expiry validation only runs where full_clean is called (forms/serializers), **not** on direct `.save()` — two different contract strengths in one model.

## Drill 6: `task_index_instance` — [dynamic_search/tasks.py L46–L89](../../../mayan/apps/dynamic_search/tasks.py#L41-L89)

Notes: inputs are strings/ints only (queue contract). `DoesNotExist` → retry (commit-lag race, not error). Unknown exception → wrapped in `DynamicSearchException` with the kwargs baked into the message (L72–L87) — poison-message diagnosis by log. Invariant: idempotent (re-indexing overwrites). Failure: max retries exhausted → task dies silently (`ignore_result=True`) — staleness with no user-visible trace.

## Drill 7: `WorkflowInstance.get_transition_choices` — [workflow_instance_models.py L171–L195](../../../mayan/apps/document_states/models/workflow_instance_models.py#L171-L195)

Notes: three stacked filters: graph (origin transitions), ACL (only if `_user` passed — `None` skips authz! trust-the-caller again), template conditions — evaluated *per row in Python* with `queryset.exclude` inside a loop (L184–L186): N template renders + up to N queryset re-creations per call. Output: queryset (lazy). Failure: condition template exceptions — where do they surface? (Follow `evaluate_condition` — good extra credit.)

## Drill 8: `FileLock._init` — [file_lock.py L62–L92](../../../mayan/apps/lock_manager/backends/file_lock.py#L62-L92)

Notes: two locks at once — a `threading.Lock` (process-local) and an OS file lock (cross-process, same host). Lock table = JSON dict in one file keyed by name; expiration enables takeover (L78–L85). Failure modes: `raise LockError` path releases the threading lock first (L84–L85) but an unexpected exception (say, corrupt JSON at L74) leaves it held → process-wide deadlock; `_release` catches `EOFError` where empty-string `json.loads` actually raises `JSONDecodeError` ([L94–L101](../../../mayan/apps/lock_manager/backends/file_lock.py#L94-L101)). Also: this backend is **host-local by nature** — correctness depends on single-host deployment.

## Drill 9: `ActionExporter.export` — [events/classes.py L41–L68](../../../mayan/apps/events/classes.py#L41-L68)

Notes: optional `user` triggers ACL restriction of the export (L50–L55) — export honors permissions, good. `iterator()` streams rows — memory-bounded. Failures/quirks: the header is written by hand with a `('\n',)` tuple concat (L60) producing a trailing comma+newline oddity vs `writer.writerow`; every value goes through `str(getattr(...))` — CSV formula-injection surface if opened in Excel (`=cmd|...` in a document label). Both are ticket material.

## Drill 10: `DocumentFile.open` + pre-open hooks — [document_file_models.py L367–L387](../../../mayan/apps/documents/models/document_file_models.py#L367-L387)

Notes: `raw=True` bypasses hooks (used by download event wrapper L286–L294 — so downloads skip pre-open transforms; is that intended for encrypted-at-rest hooks?). Hook chain may *replace* the file object (L379–L387) — a decryption layer slot. Invariant: `self.file.close()` before storage reopen — the FileField's descriptor vs storage duality. Failure: hook raising blocks every open.

**Meta-drill:** re-annotate Drill 1 after finishing module 03. Count the failure modes you see now that you didn't the first time. That delta is the curriculum working.
