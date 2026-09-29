# Review katas

Eight fake PRs against this codebase. For each: read the intent and diff summary, write your review (findings graded **Blocking / Important / Optional**), then compare. Use kind, specific language — model comments included. Timed interview versions of katas 2, 4, 6: [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).

---

## Kata 1: "Add document favorite count to the API"

**Author intent:** expose how many users favorited a document.
**Fake diff summary:** adds `favorites_count = serializers.SerializerMethodField()` to `DocumentSerializer` returning `obj.favorites.count()`.
**Files this resembles:** [document_serializers.py L23–L62](../../../mayan/apps/documents/serializers/document_serializers.py#L23-L62).
**Expected findings** — *Blocking:* N+1 — one COUNT per document on every list page; require `annotate(Count(...))` in the view queryset or a documented decision. *Important:* is favorite data private? another user's count reveals usage — check what ACL applies. *Optional:* name it `favorite_count` to match house singular style.
**Good comment:** > This adds a COUNT query per row on `/documents/` (100 rows = 100 queries). Could we annotate in `get_queryset` instead? Happy to pair on the `ModelQueryFields` hook if useful.

## Kata 2: "Retry OCR on any failure"

**Author intent:** OCR sometimes fails transiently; retry everything.
**Fake diff summary:** wraps the body of `task_document_version_page_ocr_process` in `except Exception: self.retry(exc=exception)`.
**Files this resembles:** [ocr/tasks.py L51–L90](../../../mayan/apps/ocr/tasks.py#L51-L90).
**Expected findings** — *Blocking:* corrupt-file failures now retry until max, burning worker_d and delaying the chord finisher for every healthy page; keep the existing named-exception taxonomy and add the newly-observed transient type only. *Important:* what's the evidence? ask for the error class from logs. *Optional:* mention dead-letter gap as follow-up issue.

## Kata 3: "Speed up tests by removing migration tests"

**Fake diff summary:** deletes `test_migrations.py` files; CI time −8 min.
**Expected findings** — *Blocking:* migration tests are the only executable proof that upgrade paths work ([MayanMigratorTestCase](../../../mayan/apps/testing/tests/base.py#L83-L88)); propose running them in a nightly job instead of deletion. *Important:* measure which tests are actually slow before cutting categories.

## Kata 4: "Fix stub flag on file delete"

**Author intent:** the `is_stub=False` on zero-files-left looks inverted ([document_file_models.py L231–L234](../../../mayan/apps/documents/models/document_file_models.py#L220-L236)); flip it to `True`.
**Fake diff summary:** one-line change, no test.
**Expected findings** — *Blocking:* no regression test; also **the fix may be wrong** — flipping to `True` re-exposes the document to the stub reaper ([documents/tasks.py L129–L137](../../../mayan/apps/documents/tasks.py#L129-L137)), which would *auto-delete documents whose last file was removed*. Maybe `False` is intentional (protect the shell document)? Demand: a test capturing intended behavior + a maintainer decision on semantics, not a silent flip. This kata teaches the biggest review lesson: **an obviously-wrong line can be load-bearing.**
**Good comment:** > Agreed this reads inverted vs the help text. Before flipping: `task_document_stubs_delete` reaps `is_stub=True` docs — with this change, deleting a document's last file schedules the document itself for deletion. Is that what we want? Could we write the test for the behavior we intend first?

## Kata 5: "Add `?ordering=` to a UI list view"

**Fake diff summary:** new view passes `request.GET['_ordering']` into `order_by()` directly, copying [SortingViewMixin](../../../mayan/apps/views/mixins.py#L590-L611).
**Expected findings** — *Blocking:* unvalidated user-controlled ordering — `?_ordering=user__password` style probes and guaranteed 500s on bad fields (`FieldError`); whitelist sortable fields (the API side does this via DRF's OrderingFilter with empty defaults, [rest_api/filters.py L27–L31](../../../mayan/apps/rest_api/filters.py#L27-L31)). *Important:* note the existing mixin shares the weakness — file an issue rather than propagate.

## Kata 6: "Restrict change-type target"

**Author intent:** close the asymmetry from [03/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) — require `permission_document_create` on the target type.
**Fake diff summary:** adds `AccessControlList.objects.restrict_queryset` around the serializer's `DocumentType.objects.all()` ([document_serializers.py L65–L68](../../../mayan/apps/documents/serializers/document_serializers.py#L65-L68)); updates the existing test to grant the extra permission.
**Expected findings** — *Important (not blocking!):* behavior change to a **test-enshrined** contract ([test_document_api.py L85–L112](../../../mayan/apps/documents/tests/test_document_api.py#L85-L112)) — needs a changelog entry, a release note, and maintainer sign-off; suggest a deprecation setting. *Important:* serializer needs request context to know the user — verify how `context['request']` reaches `PrimaryKeyRelatedField.queryset` (it can't be a static queryset anymore; needs a method or custom field). *Optional:* also cover the UI form path ([DocumentTypeChangeView](../../../mayan/apps/documents/views/document_views.py#L80-L90)) or the fix is API-only theater.
**Lesson:** security fixes that change documented behavior are *process* problems as much as code problems.

## Kata 7: "Cache the ACL check"

**Fake diff summary:** memoizes `restrict_queryset(permission, user)` results in a module-level dict for 60s.
**Expected findings** — *Blocking:* module-level cache is per-process (gunicorn workers × celery workers all diverge) and unbounded (querysets keyed by user × permission); grants/revocations take up to 60s to apply **per process** — a permission *revocation* delay is a security regression. *Important:* querysets are lazy — caching them caches nothing useful anyway (they re-execute); author likely wanted to cache the role-short-circuit boolean. *Optional:* point at the real hotspot list from profiling first ([05/04](../05-quality-engineering/04-performance-thinking.md)).

## Kata 8: "Watch folder: ingest all files per tick"

**Fake diff summary:** replaces the single-file `return` with a full-directory loop ([watch_folder_backends.py L100–L112](../../../mayan/apps/sources/source_backends/watch_folder_backends.py#L100-L112)).
**Expected findings** — *Blocking:* the periodic task holds one lock with `DEFAULT_SOURCES_LOCK_EXPIRE` timeout ([sources/tasks.py L24–L33](../../../mayan/apps/sources/tasks.py#L18-L55)) — a 50k-file directory now overruns the lock expiry, a second worker takes over, and both unlink files concurrently. Batch with a bound (N files or T seconds), renew or respect the lock budget. *Important:* memory — the current shape stages one file; the loop stages many before dispatching? (check ordering). *Optional:* per-tick metrics.

---

**Review-language crib (use in every kata):** lead with a question, not a verdict; quantify the failure ("100 rows = 100 queries"); anchor to precedent in-repo ("the API side whitelists via…"); separate *this PR must* from *we should file*. The full mindset: [07/01-code-review-mindset.md](../07-career-and-collaboration/01-code-review-mindset.md).

**Self-grade:** *Basic* — you found the blocking issue in 5+ katas. *Solid* — you also caught the process issues (katas 3, 6). *Strong* — kata 4: you refused to approve **either** behavior without a decision, and your comment proposed the deciding test.
