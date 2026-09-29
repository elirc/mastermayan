# Pattern Catalog

## Pattern: ACL-filtered queryset
**Problem it solves:** Prevents unauthorized object access before object lookup.
**General shape:** Filter the queryset by permission and user, then resolve the requested object from that filtered set.
**Real example:** [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L79-L86)
**Second example:** [`web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L42-L50)
**Why this implementation works:** The object is never fetched outside the allowed set.
**Failure modes:**
- fetching by PK first
- checking permission after side effects started
**Use it when:**
- user-facing selection includes object identifiers
**Avoid it when:**
- you are working on truly global resources with no object scope
**Drill:** Find another `restrict_queryset` use and explain the protected asset.

## Pattern: Staged upload handoff
**Problem it solves:** Moves file ingestion out of the request path safely.
**General shape:** Save temporary upload -> queue worker -> reopen in worker.
**Real example:** [`web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L57-L70)
**Second example:** [`sources/models.py`](../../mayan/apps/sources/models.py#L125-L161)
**Why this implementation works:** It avoids serializing the raw upload into broker messages.
**Failure modes:**
- orphaned temporary files
- worker never deletes temp row
**Use it when:** large or slow file handling exists.
**Avoid it when:** work is tiny and atomic.
**Drill:** Explain why `SharedUploadedFile` is better than passing raw bytes through Celery.

## Pattern: Create-then-rollback companion record
**Problem it solves:** Avoids leaving a broken document without its initial file.
**General shape:** create parent -> attempt child -> delete parent on child failure.
**Real example:** [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L146-L176)
**Second example:** No second example found.
**Why this implementation works:** It makes the initial document/file pair behave like a stronger unit.
**Failure modes:** cleanup can fail too.
**Use it when:** parent without child is invalid for business semantics.
**Avoid it when:** parent is intentionally allowed to be incomplete.
**Drill:** Contrast this with stub-document behavior.

## Pattern: Rich model save for derived fields
**Problem it solves:** Ensures checksum, MIME type, size, and pages stay in sync with a newly uploaded file.
**General shape:** after insert, compute derived data and persist it.
**Real example:** [`document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py#L460-L484)
**Second example:** No second example found.
**Why this implementation works:** Derived state is guaranteed close to write time.
**Failure modes:** expensive save path, hidden side effects.
**Use it when:** derived state is mandatory and local to the model.
**Avoid it when:** computation is very slow or externally dependent.
**Drill:** List two reasons to move some of this work async.

## Pattern: Thin submission, thick worker
**Problem it solves:** Keeps requests fast and workers responsible for expensive logic.
**Real example:** [`file_metadata/methods.py`](../../mayan/apps/file_metadata/methods.py#L14-L28)
**Second example:** [`document_parsing/methods.py`](../../mayan/apps/document_parsing/methods.py#L37-L49)
**Failure modes:** forgetting to pass enough context, especially user/audit context.
**Use it when:** enrichment is non-blocking.
**Avoid it when:** immediate consistency is required for the response.
**Drill:** Compare method and task signatures for metadata vs parsing.

## Pattern: Per-resource lock for async work
**Problem it solves:** Prevents duplicate concurrent processing of the same file.
**Real example:** [`file_metadata/tasks.py`](../../mayan/apps/file_metadata/tasks.py#L31-L48)
**Second example:** [`sources/tasks.py`](../../mayan/apps/sources/tasks.py#L24-L55)
**Failure modes:** lock leak, lock too broad, silent skip when lock unavailable.
**Use it when:** duplicate work is harmful.
**Avoid it when:** tasks are naturally idempotent and cheap.
**Drill:** What user-visible issue could duplicate metadata processing cause?

## Pattern: Fan-out then finish callback
**Problem it solves:** Parallelizes page-level OCR while preserving a single completion step.
**Real example:** [`ocr/tasks.py`](../../mayan/apps/ocr/tasks.py#L30-L43)
**Second example:** No second example found.
**Failure modes:** partial results, retry storms, weak visibility.
**Use it when:** subunits are independent and aggregate completion matters.
**Avoid it when:** order and transactional all-or-nothing semantics are required.
**Drill:** Name one metric you would add to this flow.

## Pattern: Event-specific delete semantics
**Problem it solves:** Records different audit meaning for user check-in, forced check-in, and auto expiration.
**Real example:** [`checkouts/models.py`](../../mayan/apps/checkouts/models.py#L74-L87)
**Second example:** [`document_models.py`](../../mayan/apps/documents/models/document_models.py#L142-L163)
**Failure modes:** collapsing distinct business events into generic delete.
**Use it when:** delete is business-significant.
**Avoid it when:** deletion is purely technical cleanup.
**Drill:** What reports become possible because of this distinction?

## Pattern: Global login-required with explicit public exceptions
**Problem it solves:** Secure-by-default route posture.
**Real example:** [`mayan/settings/base.py`](../../mayan/settings/base.py#L108-L123)
**Second example:** [`authentication_views.py`](../../mayan/apps/authentication/views/authentication_views.py#L107-L111)
**Failure modes:** accidentally exposing public views or blocking necessary auth flows.
**Use it when:** app should be private by default.
**Avoid it when:** most pages must be public.
**Drill:** List two views that must stay public.

## Pattern: Callback dotted path for post-task integration
**Problem it solves:** Lets generic upload work call source-specific post-processing.
**Real example:** [`sources/models.py`](../../mayan/apps/sources/models.py#L149-L160)
**Second example:** No second example found.
**Failure modes:** brittle dotted paths, hidden coupling.
**Use it when:** shared pipeline needs pluggable completion behavior.
**Avoid it when:** direct composition is simpler and local.
**Drill:** Identify one test that should exist for callback correctness.

## Pattern: Client-side delegated event handling
**Problem it solves:** Keeps interaction handlers working even when partial HTML is replaced.
**Real example:** [`partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L273-L280)
**Second example:** [`mayan_app.js`](../../mayan/apps/appearance/static/appearance/js/mayan_app.js#L53-L69)
**Failure modes:** duplicate handlers, event propagation surprises.
**Use it when:** AJAX rewrites DOM regions.
**Avoid it when:** component ownership is local and stable.
**Drill:** Why would direct binding break after `#ajax-content` replacement?

## Pattern: Request throttling and cancellation in AJAX navigation
**Problem it solves:** Prevents UI races and wasteful overlapping requests.
**Real example:** [`partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L96-L115)
**Second example:** No second example found.
**Failure modes:** canceled request side effects on server, stale referer assumptions.
**Use it when:** users can click rapidly through partial views.
**Avoid it when:** transport layer already guarantees request coalescing.
**Drill:** Explain `currentAjaxRequest.abort()` to an interviewer.
