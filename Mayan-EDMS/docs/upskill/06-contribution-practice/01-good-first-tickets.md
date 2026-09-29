# Good First Tickets

## Ticket 1: Add docs note for upload temp-file lifecycle
**Difficulty:** Easy
**Estimated time:** 45-90 min
**Skills practiced:** tracing async flows, technical writing
**Story:** As a contributor, I want the upload path documented so I do not break temp-file cleanup accidentally.
**Why this is a good contribution:** Useful, low blast radius, grounded in real code.
**Acceptance criteria:**
- [ ] docs mention `SharedUploadedFile` purpose
- [ ] docs mention worker cleanup point
**Read these anchors first:**
- [`web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L57-L70)
- [`sources/tasks.py`](../../mayan/apps/sources/tasks.py#L95-L103)
**Files likely touched:** docs only
**Implementation plan:** trace lifecycle, then document verified behavior only.
**What could go wrong:** overstating guarantees not verified in code.
**Suggested checks:** re-read request and task files after writing.
**Review questions:** Does the note distinguish verified behavior from inference?

## Ticket 2: Add a permission-denied API test for document upload
**Difficulty:** Easy
**Estimated time:** 1-2 hours
**Skills practiced:** DRF testing, ACL reasoning
**Story:** As a maintainer, I want upload permission regressions caught early.
**Why this is a good contribution:** Fits existing test patterns.
**Acceptance criteria:**
- [ ] a denied case is covered
- [ ] no document is created on denial
**Read these anchors first:**
- [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L119-L129)
- [`mayan/apps/documents/tests/test_document_api.py`](../../mayan/apps/documents/tests/test_document_api.py)
**Files likely touched:** `mayan/apps/documents/tests/test_document_api.py`
**Implementation plan:** copy nearest upload test, remove permission grant, assert denial.
**What could go wrong:** testing the wrong permission edge.
**Suggested checks:** targeted test run.
**Review questions:** Does it fail for the intended reason?

## Ticket 3: Document the checkout invariant
**Difficulty:** Easy
**Estimated time:** 45 min
**Skills practiced:** reading model invariants
**Story:** As a contributor, I want checkout rules explained before I bypass them.
**Why this is a good contribution:** low-risk clarity win.
**Acceptance criteria:**
- [ ] note explains one-checkout-per-document
- [ ] note mentions expiration or event semantics
**Read these anchors first:** [`checkouts/models.py`](../../mayan/apps/checkouts/models.py#L68-L116)

## Ticket 4: Add a docs note about delegated JS event handlers
**Difficulty:** Easy
**Estimated time:** 30-60 min
**Skills practiced:** legacy front-end reading
**Story:** As a learner, I want to understand why delegated handlers exist here.
**Why this is a good contribution:** teaches a transferable concept with low risk.
**Acceptance criteria:** mention DOM replacement and delegated events.
**Read these anchors first:** [`partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L273-L280)

## Ticket 5: Add a test for `DocumentType.new_document()` rollback on file failure
**Difficulty:** Medium
**Estimated time:** 2-4 hours
**Skills practiced:** failure injection, model lifecycle tests
**Story:** As a maintainer, I want parent cleanup enforced if initial file creation fails.
**Why this is a good contribution:** protects a meaningful invariant.
**Acceptance criteria:**
- [ ] force the child-file step to fail
- [ ] assert parent document is removed
**Read these anchors first:** [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L146-L176)

## Ticket 6: Improve verification notes around missing local dependencies
**Difficulty:** Easy
**Estimated time:** 30 min
**Skills practiced:** contributor empathy, docs
**Story:** As a new contributor, I want setup failures explained faster.
**Why this is a good contribution:** common newcomer pain point, tiny blast radius.

## Ticket 7: Add a test for `DocumentCheckout.clean()` past-date rejection
**Difficulty:** Easy
**Estimated time:** 1 hour
**Skills practiced:** model validation tests
**Story:** As a maintainer, I want expiration validation guarded by a regression test.
**Why this is a good contribution:** simple invariant coverage.
**Read these anchors first:** [`checkouts/models.py`](../../mayan/apps/checkouts/models.py#L68-L72)

## Ticket 8: Add a small docs page comparing parsing vs OCR
**Difficulty:** Easy
**Estimated time:** 1 hour
**Skills practiced:** concept contrast, code anchoring
**Story:** As a learner, I want to stop confusing text parsing with OCR.
**Why this is a good contribution:** directly helpful for juniors.

## Ticket 9: Add a test that upload API still ACL-filters document types
**Difficulty:** Medium
**Estimated time:** 2 hours
**Skills practiced:** security regression testing
**Story:** As a maintainer, I want the authorization boundary pinned by tests.
**Why this is a good contribution:** security-focused and plausible.
**Read these anchors first:**
- [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L119-L129)
- [`web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L42-L50)

## Ticket 10: Add a docs table mapping major Celery tasks to triggering actions
**Difficulty:** Easy
**Estimated time:** 45 min
**Skills practiced:** async cartography
**Story:** As a learner, I want one page that maps user action to task fan-out.
**Why this is a good contribution:** useful and low-risk.

## Ticket 11: Add a JS interview-prep note explaining AJAX request cancellation
**Difficulty:** Easy
**Estimated time:** 30 min
**Skills practiced:** JavaScript runtime explanation
**Story:** As a learner, I want to turn legacy jQuery code into interview material.
**Why this is a good contribution:** helps transfer repo knowledge outward.

## Ticket 12: Add a docs warning about archive expansion cost
**Difficulty:** Easy
**Estimated time:** 45 min
**Skills practiced:** performance/security awareness
**Story:** As a contributor, I want the upload recursion cost called out before changing it.
**Why this is a good contribution:** teaches judgment without product churn.

## Ticket 13: Add a regression test around OCR finish error logging
**Difficulty:** Medium
**Estimated time:** 2-4 hours
**Skills practiced:** async failure testing
**Story:** As a maintainer, I want finish-path OCR failures to remain visible.
**Why this is a good contribution:** observability protection.
**Read these anchors first:** [`ocr/tasks.py`](../../mayan/apps/ocr/tasks.py#L92-L131)

## Ticket 14: Add a docs note about global login-required defaults
**Difficulty:** Easy
**Estimated time:** 30 min
**Skills practiced:** security model explanation
**Story:** As a learner, I want to understand why auth views are special cases.
**Why this is a good contribution:** short, important, easy to review.

## Ticket 15: Add a command-cheatsheet entry for migration-only test runs
**Difficulty:** Easy
**Estimated time:** 20 min
**Skills practiced:** tooling literacy
**Story:** As a contributor, I want the migration-only test command easy to find.
**Why this is a good contribution:** tiny but practical.
