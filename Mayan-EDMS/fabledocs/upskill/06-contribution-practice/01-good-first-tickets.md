# Good first tickets

Twenty junior tickets. Each is small, anchored, and follows an existing pattern. Difficulty: Easy unless noted. Do the reading anchors *first*.

Format is compressed after ticket 1 (which shows the full template) to keep this usable.

---

## Ticket 1: Fix unreachable `except OperationalError` in the OCR finisher

**Difficulty:** Easy — **~30 min**
**Skills:** exception ordering, Celery, regression testing.
**Story:** As a maintainer, I want dead code removed so error handling reads truthfully.
**Why good contribution:** real defect, zero behavior risk, teaches exception MRO.
**Acceptance criteria:**
- [ ] `except OperationalError` in `task_document_version_ocr_finished` is reachable or removed.
- [ ] A comment or test documents intended behavior.
**Read first:** [ocr/tasks.py L114–L131](../../../mayan/apps/ocr/tasks.py#L92-L131) — `except Exception` precedes `except OperationalError`, making the latter unreachable.
**Files likely touched:** `ocr/tasks.py`.
**Implementation plan:** reorder so `OperationalError` (retry) precedes the general `Exception` (log + error row), matching the per-page task's ordering ([L80–L90](../../../mayan/apps/ocr/tasks.py#L74-L90)).
**Fake-code shape:**
```python
# Illustrative fake code: not from this repo
try:
    ...
except OperationalError as e:   # specific first → retry
    raise self.retry(exc=e)
except Exception as e:          # general last → log + error row
    document_version.ocr_errors.create(result=e); raise
```
**What could go wrong:** the two handlers have different effects (retry vs record); reordering changes behavior for OperationalError — that's the *point*, but state it in the PR.
**Suggested checks / review questions:** "which errors should retry vs record?"; add a test that raises OperationalError and asserts retry.
**Interview story potential:** "I found and fixed an unreachable exception handler — here's how exception resolution order works."

## Ticket 2: Correct the `ignore_results` typo

**Easy — 15 min.** [documents/tasks.py L140](../../../mayan/apps/documents/tasks.py#L140-L145) uses `ignore_results=True` (should be `ignore_result`). Compare correct usage two functions up ([L129](../../../mayan/apps/documents/tasks.py#L129-L137)). Verify effect (results currently stored) and fix. **Reject risk:** changing result behavior could affect anything reading task results — confirm nothing does. **Story potential:** "config kwargs that fail silently in dynamic languages."

## Ticket 3: Write a regression test pinning the `is_stub` semantics on last-file delete

**Medium — 1 hr.** [document_file_models.py L231–L234](../../../mayan/apps/documents/models/document_file_models.py#L220-L236) sets `is_stub=False` when the last file is deleted, which reads inverted vs the help text ([document_models.py L98–L104](../../../mayan/apps/documents/models/document_models.py#L98-L104)). **Do not "fix" it** — write a test capturing *current* behavior + a code comment stating the intended invariant, and open a discussion issue (kata 4 in [04/04](../04-code-reading-gym/04-review-katas.md) explains why flipping it is dangerous). **Story potential:** "an obviously-wrong line that was load-bearing."

## Ticket 4: Add `assertNumQueries` to the document list API test

**Medium — 1 hr.** Pin the query count for `APIDocumentListView` with 1 and 10 documents so future N+1 regressions fail loudly. Read [04/perf drill](../05-quality-engineering/04-performance-thinking.md) and [test_document_api.py](../../../mayan/apps/documents/tests/test_document_api.py). **Reject risk:** brittle if count includes setup queries — scope to the request. **Story potential:** "I turned a performance assumption into a test."

## Ticket 5: CSV-injection hardening in event export

**Medium — 1 hr.** Prefix leading `= + - @` in exported cell values ([events/classes.py L62–L68](../../../mayan/apps/events/classes.py#L41-L68)). Follow OWASP guidance. **Reject risk:** maintainer may deem it out of scope (admin-only export) — lead the PR with that discussion. **Story potential:** the CSV-injection security story ([08/06](../08-interview-prep/06-behavioral-star-stories.md)).

## Ticket 6: Rename misleading `document_type_id` local to reflect it's an instance

**Easy — 30 min.** In `APIDocumentChangeTypeView.object_action`, `document_type_id` is actually a DocumentType instance ([document_api_views.py L106–L110](../../../mayan/apps/documents/api_views/document_api_views.py#L95-L111)). Rename for readability (local only — not the serializer field, which is a contract). **Reject risk:** touching the serializer field name would break the API. **Story potential:** "names as documentation in an untyped codebase."

## Ticket 7: Add a docstring + type note to `restrict_queryset`

**Easy — 30 min.** The single most important security function has terse docs ([acls/managers.py L268–L294](../../../mayan/apps/acls/managers.py#L268-L294)). Add a docstring: inputs, the anonymous/role/ACL branches, and the 404-not-403 consequence. Pure docs. **Story potential:** "documenting the security boundary I had to understand."

## Ticket 8: Guard `Cache.get_total_size_display` against division by zero

**Easy — 30 min.** `get_total_size() / self.maximum_size * 100` ([file_caching/models.py L92–L96](../../../mayan/apps/file_caching/models.py#L92-L99)) — `maximum_size` is validated ≥1 at the field ([L44–L47](../../../mayan/apps/file_caching/models.py#L44-L48)), so this is a defensive/readability ticket; verify the validator truly forbids 0 and add a test. **Reject risk:** if the validator already guarantees it, a maintainer may prefer a comment over a guard — argue both.

## Ticket 9: Add missing index-consideration comment on ACL lookup

**Easy — 30 min.** Document (comment only) the `(content_type, object_id, role)` access path in [acls/models.py](../../../mayan/apps/acls/models.py) and note it as a measure-first hotspot. Pairs with [04/perf](../05-quality-engineering/04-performance-thinking.md).

## Ticket 10: Test the checkout race exception type

**Medium — 1 hr.** `DocumentCheckout.save` can raise `DocumentAlreadyCheckedOut` *or* `IntegrityError` depending on race timing ([checkouts/models.py L100–L116](../../../mayan/apps/checkouts/models.py#L100-L116)). Add a test documenting the non-raced path; comment the raced path. **Story potential:** "check-then-act vs DB constraints."

## Ticket 11: Add `no_permission` test for an endpoint missing its pair

**Medium — 1 hr.** Audit one app (e.g. `web_links` or `tags`) for any mutating endpoint lacking the `no_permission` → 404 test; add it. Follow [test_document_api.py L27–L41](../../../mayan/apps/documents/tests/test_document_api.py#L27-L41). **Story potential:** "closing an authorization test gap."

## Ticket 12: Fix the `FileLock._release` wrong exception type

**Medium — 1.5 hr.** `except EOFError` should be `except json.JSONDecodeError` (empty file case) ([file_lock.py L98–L101](../../../mayan/apps/lock_manager/backends/file_lock.py#L94-L101)). **Reject risk:** the whole file-lock backend is host-local and less-used in prod (Redis default) — maintainer may want a broader fix; scope tightly and note the thread-lock leak as a separate issue. **Story potential:** distributed-locks debugging.

## Ticket 13: Document the four worker tiers

**Easy — 45 min.** Add a docstring/README section mapping worker A–D to their latency classes and example tasks ([workers.py L12–L36](../../../mayan/apps/task_manager/workers.py#L12-L36) + queue bindings). Docs only.

## Ticket 14: Add a search-lag note to the search test helper

**Easy — 30 min.** Document (comment) in a search test that indexing is async and tests must set index state explicitly ([dynamic_search handlers](../../../mayan/apps/dynamic_search/handlers.py#L116-L125)). Prevents the next person's flaky test.

## Ticket 15: Validate watch-folder `folder_path` is inside an allowed root (design-first)

**Medium — 2 hr, design note required.** Currently any server path is accepted ([watch_folder_backends.py L88–L96](../../../mayan/apps/sources/source_backends/watch_folder_backends.py#L77-L112)). Propose (in an issue first) an allow-list setting; implement validation in the backend's `clean`. **Reject risk:** changes admin UX; needs maintainer buy-in — this is really a mid-level ticket wearing junior clothes.

## Ticket 16: Add reap-count logging to `task_document_stubs_delete`

**Easy — 30 min.** Log how many stubs were deleted ([documents/tasks.py L129–L137](../../../mayan/apps/documents/tasks.py#L129-L137)) so a broken pipeline is visible in logs. Pairs with [06-observability](../05-quality-engineering/06-observability-and-operations.md).

## Ticket 17: Fix the export header trailing-comma quirk

**Easy — 30 min.** `file_object.write(','.join(self.field_names + ('\n',)))` produces a trailing comma before newline ([events/classes.py L60](../../../mayan/apps/events/classes.py#L57-L61)); use `writer.writerow(self.field_names)` for consistency with the body. Add a test asserting header shape.

## Ticket 18: Document the `_event_actor` smuggling convention

**Easy — 45 min.** One reference doc listing where `_event_*` attributes are set and consumed ([api_view_mixins.py L153–L172](../../../mayan/apps/rest_api/api_view_mixins.py#L153-L172), [events/classes.py L161–L184](../../../mayan/apps/events/classes.py#L161-L184)). Saves every future contributor a day.

## Ticket 19: Add a bound to archive `expand` recursion depth

**Medium — 2 hr, design note.** `file_new(expand=True)` recurses on nested archives with only a code comment as a guard ([document_models.py L205–L224](../../../mayan/apps/documents/models/document_models.py#L189-L251)). Propose a max-depth setting; implement. **Reject risk:** behavior change — issue first. **Story potential:** zip-bomb DoS mitigation.

## Ticket 20: Add a `test_trashed_*` third member where missing

**Medium — 1 hr.** Find a resource with `no_permission`/`with_access` but no trashed-variant test and add it, following [test_document_api.py L114–L137](../../../mayan/apps/documents/tests/test_document_api.py#L114-L137). Teaches the valid-manager boundary.

---

**How to pick your first:** tickets 1, 2, 17 are safe morale-builders (real defects, no behavior risk). Tickets 3, 10, 12 teach the deepest lessons. Tickets 15, 19 are traps disguised as easy — attempting them teaches you to recognize "this needs a design discussion first," which is itself a promotion-worthy skill. Whatever you pick, write the PR description using [07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md) before you touch code.
