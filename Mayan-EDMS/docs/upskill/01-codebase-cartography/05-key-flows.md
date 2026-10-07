# Key Flows

## Flow: Web form upload to document creation
**Why this flow matters:** It is the shortest route from user intent to durable business state. It also shows ACL checks, task handoff, rollback, derived metadata, and signal-based follow-up in one path.
**Open these files first:**
- [`mayan/apps/sources/source_backends/web_form_backends.py`](../../../mayan/apps/sources/source_backends/web_form_backends.py#L28-L72) - request-side permission gate and task enqueue.
- [`mayan/apps/sources/tasks.py`](../../../mayan/apps/sources/tasks.py#L58-L103) - worker-side shared upload handling and retry on DB operational errors.
- [`mayan/apps/sources/models.py`](../../../mayan/apps/sources/models.py#L87-L161) - async callback indirection and archive expansion.
- [`mayan/apps/documents/models/document_type_models.py`](../../../mayan/apps/documents/models/document_type_models.py#L138-L176) - document creation plus rollback if first file creation fails.
- [`mayan/apps/documents/models/document_file_models.py`](../../../mayan/apps/documents/models/document_file_models.py#L428-L505) - checksum, mimetype, page count, signals.
**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Source backend | `web_form_backends.py` | Restrict `DocumentType` queryset by ACL | `document_type_id`, uploaded file | Permission bypass if filter removed |
| 2 | Source backend | `web_form_backends.py` | Persist `SharedUploadedFile` and queue task | temp upload row id | orphaned temp uploads |
| 3 | Celery task | `sources/tasks.py` | reload `DocumentType`, `SharedUploadedFile`, `Source`, `User` | DB ids | retry only covers `OperationalError` |
| 4 | Source model | `sources/models.py` | enqueue `task_document_upload` via callback-aware wrapper | callback metadata | callback drift |
| 5 | Document type | `document_type_models.py` | create `Document`, then first file | `Document`, `DocumentFile` | partial create |
| 6 | Document file save | `document_file_models.py` | compute derived fields, pages, signals | storage file, checksum, pages | expensive synchronous work in save |
**Validation and authorization:**
ACL filtering occurs before upload acceptance in [`web_form_backends.py`](../../../mayan/apps/sources/source_backends/web_form_backends.py#L42-L50). The API upload path applies the same idea in [`document_api_views.py`](../../../mayan/apps/documents/api_views/document_api_views.py#L119-L129).
**Persistence and side effects:**
The request persists a `SharedUploadedFile` first [`web_form_backends.py`](../../../mayan/apps/sources/source_backends/web_form_backends.py#L57-L70). Document and file persistence then occur in separate steps with explicit cleanup on failure [`document_type_models.py`](../../../mayan/apps/documents/models/document_type_models.py#L163-L174). Post-save emits `signal_post_document_file_upload` and possibly `signal_post_document_created` [`document_file_models.py`](../../../mayan/apps/documents/models/document_file_models.py#L492-L500).
**Tests that cover it:**
- [`mayan/apps/sources/tests/test_web_form_source_api.py`](../../../mayan/apps/sources/tests/test_web_form_source_api.py)
- [`mayan/apps/sources/tests/test_web_form_source_views.py`](../../../mayan/apps/sources/tests/test_web_form_source_views.py)
**What juniors usually miss:**
- The upload request does not directly create the final document file.
- Permission is checked against `DocumentType`, not only against `Source`.
- `DocumentFile.save()` performs meaningful domain work, not just persistence.
**What seniors notice:**
- Save hooks and signals make this path powerful but also implicit.
- Archive expansion can amplify work and deserves quota/rate thinking.
- Shared-upload cleanup depends on the happy path reaching deletion.
**Drill:**
Trace what happens differently when `expand=True` and the upload is a zip.
**Self-grade:**
- Basic: can name the files involved.
- Solid: can explain why `SharedUploadedFile` exists.
- Strong: can identify failure boundaries and suggest one regression test.

## Flow: API document create/upload
**Why this flow matters:** It shows the public contract exposed to external clients and how Mayan keeps API permissions aligned with browser behavior.
**Open these files first:**
- [`mayan/apps/documents/urls.py`](../../../mayan/apps/documents/urls.py#L522-L564) - API route registration.
- [`mayan/apps/documents/api_views/document_api_views.py`](../../../mayan/apps/documents/api_views/document_api_views.py#L64-L135) - list/create/upload views.
- [`mayan/apps/rest_api/api_view_mixins.py`](../../../mayan/apps/rest_api/api_view_mixins.py#L153-L172) - extra context and instance data injection.
**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Router | `documents/urls.py` | route `/documents/upload/` to `APIDocumentUploadView` | HTTP request | contract break if route changes |
| 2 | API view | `document_api_views.py` | restrict `DocumentType` by ACL | `document_type_id` | IDOR if restriction removed |
| 3 | Serializer/create path | serializer + model | create document and file | request payload | validation drift |
**Validation and authorization:**
`APIDocumentUploadView.perform_create()` restricts document types with `AccessControlList.objects.restrict_queryset(...)` before resolving the selected type [`document_api_views.py`](../../../mayan/apps/documents/api_views/document_api_views.py#L119-L129).
**Persistence and side effects:**
Uses the same document/domain models as the browser flow, so the same save hooks and signals apply.
**Tests that cover it:**
- [`mayan/apps/documents/tests/test_document_api.py`](../../../mayan/apps/documents/tests/test_document_api.py)
**What juniors usually miss:**
- API and browser routes share models but not necessarily identical request shapes.
**What seniors notice:**
- The public contract is the route + serializer + permission map, not just the model.
**Drill:**
Compare API upload with source upload. What responsibilities are duplicated, and what responsibilities are intentionally different?
**Self-grade:**
- Basic: names endpoint and permission.
- Solid: explains why `document_type_id` is revalidated server-side.
- Strong: identifies one contract test to protect backward compatibility.

## Flow: Object-level authorization with ACLs
**Why this flow matters:** Mayan EDMS is multi-user and permission-heavy. If you do not understand ACL filtering, you will propose unsafe changes.
**Open these files first:**
- [`mayan/apps/acls/models.py`](../../../mayan/apps/acls/models.py#L22-L118) - ACL data model.
- [`mayan/apps/documents/models/document_type_models.py`](../../../mayan/apps/documents/models/document_type_models.py#L114-L120) - ACL-restricted document counting.
- [`mayan/apps/documents/api_views/document_api_views.py`](../../../mayan/apps/documents/api_views/document_api_views.py#L79-L86) - API enforcement example.
**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | ACL model | `acls/models.py` | represent role/object/permission tuple | content type, object id, role | incorrect uniqueness assumptions |
| 2 | API or domain code | various | restrict queryset by permission and user | queryset | overbroad data access |
| 3 | consumer | views/serializers | operate only on filtered objects | model instance | hidden authorization dependency |
**Validation and authorization:**
Authorization is not centralized in one middleware. It is repeatedly enforced via queryset restriction and method-level permission declarations.
**Persistence and side effects:**
ACL permission add/remove commits edit events [`acls/models.py`](../../../mayan/apps/acls/models.py#L88-L104).
**Tests that cover it:**
- [`mayan/apps/documents/tests/test_permissions.py`](../../../mayan/apps/documents/tests/test_permissions.py)
- [`mayan/apps/cabinets/tests/test_api.py`](../../../mayan/apps/cabinets/tests/test_api.py)
**What juniors usually miss:**
- Looking up by primary key before ACL restriction is an IDOR trap.
**What seniors notice:**
- Generic foreign keys trade type safety for flexibility; test coverage becomes more important.
**Drill:**
Find two more uses of `restrict_queryset` and describe what object is being protected.
**Self-grade:**
- Basic: explains what an ACL row stores.
- Solid: explains why queryset filtering is safer than "check after fetch."
- Strong: spots one place where cross-object permission coordination is required.

## Flow: Document file metadata extraction
**Why this flow matters:** It is a clean example of event submission, lock-based concurrency control, and async enrichment that does not block user requests.
**Open these files first:**
- [`mayan/apps/file_metadata/methods.py`](../../../mayan/apps/file_metadata/methods.py#L7-L28) - submit method.
- [`mayan/apps/file_metadata/tasks.py`](../../../mayan/apps/file_metadata/tasks.py#L17-L48) - worker logic.
**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | document/document file method | `methods.py` | submit latest file or one file | `document_file_id`, `user_id` | no-op confusion on stub docs |
| 2 | worker | `tasks.py` | acquire per-file lock | lock id string | duplicate work if lock missing |
| 3 | driver | `tasks.py` | process metadata drivers | document file | slow driver or driver exception |
**Validation and authorization:**
This flow assumes authorization happened before submission; task execution itself trusts IDs.
**Persistence and side effects:**
Enrichment is persisted by the driver layer; task layer mainly schedules and guards concurrency.
**Tests that cover it:**
- [`mayan/apps/file_metadata/tests/test_classes.py`](../../../mayan/apps/file_metadata/tests/test_classes.py)
- [`mayan/apps/file_metadata/tests/test_indexing.py`](../../../mayan/apps/file_metadata/tests/test_indexing.py)
**What juniors usually miss:**
- Locks are about correctness, not only performance.
**What seniors notice:**
- The lock is coarse per file, which is usually right here because drivers are side-effectful.
**Drill:**
Explain why `user_id` is passed to the task instead of a full user object.
**Self-grade:**
- Basic: identifies the task.
- Solid: explains the lock purpose.
- Strong: proposes one failure-mode test.

## Flow: Parsing extracted text from document files
**Why this flow matters:** Searchability and text introspection depend on parsing. This is also a good contrast with OCR: parsing operates on files, OCR on pages/images.
**Open these files first:**
- [`mayan/apps/document_parsing/methods.py`](../../../mayan/apps/document_parsing/methods.py#L16-L49)
- [`mayan/apps/document_parsing/tasks.py`](../../../mayan/apps/document_parsing/tasks.py#L11-L34)
**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | method | `methods.py` | submit latest file for parsing | document file id | stale "latest file" assumptions |
| 2 | task | `tasks.py` | load file and call page-content manager | document file | manager-side exceptions |
**Validation and authorization:**
Authorization is out-of-band here; task starts after an already-authorized action.
**Persistence and side effects:**
Page content is persisted through the manager invoked in the task.
**Tests that cover it:**
- Search and parsing tests under [`mayan/apps/document_parsing/tests/`](../../../mayan/apps/document_parsing/tests/)
**What juniors usually miss:**
- Parsing content is not the same as file metadata and not the same as OCR.
**What seniors notice:**
- This is a natural place to ask about idempotency and stale content invalidation when a new file supersedes the old one.
**Drill:**
Compare the parsing task signature to the file metadata task signature. Why are they similar?
**Self-grade:**
- Basic: can distinguish parsing from OCR.
- Solid: can explain why tasks reload objects by PK.
- Strong: identifies one invalidation concern.

## Flow: OCR fan-out and finish callback
**Why this flow matters:** It is the repo’s clearest async orchestration example: split by page, retry page work, then emit a finish event.
**Open these files first:**
- [`mayan/apps/ocr/tasks.py`](../../../mayan/apps/ocr/tasks.py#L17-L131) - page fan-out, retries, finish.
**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | version task | `ocr/tasks.py` | build one task per page | page ids | large fan-out |
| 2 | page task | `ocr/tasks.py` | process OCR content | page id | cache miss, lock error, DB error |
| 3 | finish task | `ocr/tasks.py` | clear error log and emit event | document version id | partial completion ambiguity |
**Validation and authorization:**
Triggered after an allowed action; tasks rely on internal trust boundaries.
**Persistence and side effects:**
Writes OCR content and error logs, then emits `event_ocr_document_version_finished` [`ocr/tasks.py`](../../../mayan/apps/ocr/tasks.py#L114-L125).
**Tests that cover it:**
- [`mayan/apps/ocr/tests/`](../../../mayan/apps/ocr/tests/)
**What juniors usually miss:**
- Retryable exceptions are selective.
- Success cleanup is explicit, not automatic.
**What seniors notice:**
- A chord introduces coordination semantics; partial task success requires careful observability.
**Drill:**
List every retryable exception in the page task and explain what system pressure each implies.
**Self-grade:**
- Basic: explains the page fan-out.
- Solid: identifies finish callback responsibilities.
- Strong: proposes one metric to detect OCR degradation.

## Flow: Document checkout invariant
**Why this flow matters:** It is a compact example of domain rules living in a model instead of in a controller.
**Open these files first:**
- [`mayan/apps/checkouts/models.py`](../../../mayan/apps/checkouts/models.py#L28-L116)
**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | model validation | `checkouts/models.py` | future expiration required | datetime | invalid lease |
| 2 | save invariant | `checkouts/models.py` | reject duplicate checkout | document relation | race conditions |
| 3 | delete eventing | `checkouts/models.py` | different event based on actor | checkout row | incorrect audit trail |
**Validation and authorization:**
Validation happens in `clean()` and `save()`; authorization is elsewhere.
**Persistence and side effects:**
Save emits checkout event; delete chooses between checked-in, forceful, or auto event types.
**Tests that cover it:**
- [`mayan/apps/checkouts/tests/`](../../../mayan/apps/checkouts/tests/)
**What juniors usually miss:**
- `OneToOneField` is not the only invariant; save logic still matters.
**What seniors notice:**
- Without DB-side transactional locking, concurrent save attempts deserve scrutiny.
**Drill:**
Explain why the event type is decided at delete time instead of later in reporting code.
**Self-grade:**
- Basic: names the invariant.
- Solid: connects the invariant to user-visible behavior.
- Strong: identifies a concurrency test worth adding.

## Flow: Login and multi-factor escalation
**Why this flow matters:** It shows authentication flow control, session handoff, and public-route exceptions in a mostly login-required system.
**Open these files first:**
- [`mayan/apps/authentication/django_authentication_backends.py`](../../../mayan/apps/authentication/django_authentication_backends.py#L7-L21) - email backend and timing defense.
- [`mayan/apps/authentication/views/authentication_views.py`](../../../mayan/apps/authentication/views/authentication_views.py#L45-L145) - MFA wizard.
- [`mayan/settings/base.py`](../../../mayan/settings/base.py#L108-L123) - `LoginRequiredMiddleware` and CSRF middleware.
**Trace:**
| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | backend | `django_authentication_backends.py` | load user by email | username/password | user enumeration timing |
| 2 | login view | `authentication_views.py` | branch to direct login or MFA wizard | cleaned credentials | inconsistent session state |
| 3 | MFA wizard | `authentication_views.py` | process forms, retrieve user, log in | session user id | incomplete factor flow |
**Validation and authorization:**
Public access is explicit via `StrongholdPublicMixin` and `@public` on auth views.
**Persistence and side effects:**
Writes session state and may send password-reset emails in the reset flow.
**Tests that cover it:**
- [`mayan/apps/authentication/tests/test_login_views.py`](../../../mayan/apps/authentication/tests/test_login_views.py)
- [`mayan/apps/authentication_otp/tests/test_views.py`](../../../mayan/apps/authentication_otp/tests/test_views.py)
**What juniors usually miss:**
- The app defaults to login-required globally; auth pages are exceptions.
**What seniors notice:**
- The fallback password hash call is a concrete anti-enumeration measure.
**Drill:**
Explain the purpose of `SESSION_MULTI_FACTOR_USER_ID_KEY`.
**Self-grade:**
- Basic: can describe login vs MFA branch.
- Solid: mentions the timing-attack mitigation.
- Strong: identifies one session-state regression test.

## Verification Notes

- These flows were built from inspected code, not only test names.
- I did not run worker processes or a full app instance in this environment.
