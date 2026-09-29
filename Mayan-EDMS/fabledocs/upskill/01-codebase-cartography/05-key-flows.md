# Key flows

Eight end-to-end traces. Every other module reuses these, rotating among them — learn them here once. Line anchors verified against v4.3.1.

---

## Flow 1: API document upload (`POST /api/v4/documents/upload/`)

Why this flow matters: it crosses HTTP → validation → authorization → DB → queue → worker → storage, and shows the repo's signature move: *return early, finish in a worker*.

Open these files first:
- [document_api_views.py](../../../mayan/apps/documents/api_views/document_api_views.py#L113-L135) — the view
- [document_serializers.py](../../../mayan/apps/documents/serializers/document_serializers.py#L71-L99) — the serializer that dispatches the task
- [documents/tasks.py](../../../mayan/apps/documents/tasks.py#L52-L124) — the worker half
- [document_models.py](../../../mayan/apps/documents/models/document_models.py#L189-L251) — `file_new`

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | DRF view | [api view L113–130](../../../mayan/apps/documents/api_views/document_api_views.py#L113-L130) | `perform_create` ACL-restricts DocumentType queryset by `permission_document_create`, 404s if type not granted | multipart: `file`, `document_type_id`, `label?` | authz by 404, not 403 (existence hiding) |
| 2 | Serializer | [serializers L74–80](../../../mayan/apps/documents/serializers/document_serializers.py#L74-L80) | Creates `Document` row (still a stub), parks bytes as `SharedUploadedFile` | Document.pk + SharedUploadedFile.pk | two rows now exist that a crash could orphan |
| 3 | Serializer | [serializers L82–90](../../../mayan/apps/documents/serializers/document_serializers.py#L82-L90) | `task_document_file_upload.apply_async(document_id, shared_uploaded_file_id, user_id)` | primitive IDs only | task is fire-and-forget; API returns before file exists |
| 4 | Worker (queue `uploads`, worker_c) | [tasks L64–80](../../../mayan/apps/documents/tasks.py#L64-L80) | Re-fetches rows; `OperationalError` → `self.retry` | | DB hiccup handled; *missing rows are not* — raises |
| 5 | Worker | [tasks L82–96](../../../mayan/apps/documents/tasks.py#L82-L96) | Opens staged file, calls `document.file_new(...)`; a `Warning` (e.g. checked-out block) deletes staging quietly | file object | checkout veto arrives via `Warning`, an unusual control channel |
| 6 | Model | [document_models L230–251](../../../mayan/apps/documents/models/document_models.py#L230-L251) | `DocumentFile` created → full `save()` pipeline (Flow 4) → post-action sets version pages | DocumentFile row + pages | see Flow 4 |
| 7 | Worker | [tasks L117–124](../../../mayan/apps/documents/tasks.py#L117-L124) | Deletes `SharedUploadedFile` in success path (and in error paths above) | | cleanup failure leaves orphaned staging blobs |

Validation and authorization: DRF field validation in the serializer; type-level ACL in `perform_create` ([L119–L130](../../../mayan/apps/documents/api_views/document_api_views.py#L119-L130)). Note the **create-only** guard: `document_type_id` is in `create_only_fields` ([serializers L44](../../../mayan/apps/documents/serializers/document_serializers.py#L44)) so PATCH can't sneak a type change past the weaker edit permission.

Persistence and side effects: rows in steps 2/6; storage write in Flow 4; events `document_created` / `document_file_created` from model decorators.

Tests that cover it: [test_document_upload_api.py](../../../mayan/apps/documents/tests/) exercises the endpoint; the house pattern is a `..._no_permission` / `..._with_access` pair per view (see [test_document_api.py L27–L66](../../../mayan/apps/documents/tests/test_document_api.py#L27-L66)).

What juniors usually miss: the API answers **before** the file is processed — the client owns polling for readiness (`is_stub`, page count). Also that the checksum is computed by the *worker*, so duplicate detection can't happen at request time.

What seniors notice: no idempotency key — a client retrying step 3's HTTP call after a network timeout creates a second document; contrast with the task-side retry which *is* safe because `OperationalError` retries re-run a not-yet-committed unit. Also the `Warning`-as-veto channel in step 5: clever, but invisible to API clients (silent success-shaped failure).

Interview angle: "Design a file-upload API" — this flow is your worked example: 202-style async, staging table, ID-passing, cleanup discipline. Card [Q4 in 03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: the task in step 4 retries on `OperationalError` with `UPLOAD_NEW_VERSION_RETRY_DELAY`. Find what happens if `document.file_new` raises a non-OperationalError exception mid-write. Self-grade — *Basic:* staging file is deleted in the `except` branch ([tasks L103–L116](../../../mayan/apps/documents/tasks.py#L103-L116)). *Solid:* you note the document row remains a stub and gets reaped by `task_document_stubs_delete`. *Strong:* you can argue whether deleting the staging file on unknown errors is right (it forecloses retry) and propose a dead-letter alternative.

---

## Flow 2: Watch-folder ingestion (periodic, no human)

Why this flow matters: a full background pipeline with a **distributed lock**, per-tick work limits, and destructive side effects on external state.

Open these files first:
- [sources/tasks.py](../../../mayan/apps/sources/tasks.py#L18-L55) — the periodic task with the lock
- [watch_folder_backends.py](../../../mayan/apps/sources/source_backends/watch_folder_backends.py#L77-L112) — the scan
- [sources mixins](../../../mayan/apps/sources/source_backends/mixins.py#L95-L120) — `process_documents`
- [sources/tasks.py](../../../mayan/apps/sources/tasks.py#L58-L103) — `task_process_document_upload`

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | beat → worker_c | [tasks L24–33](../../../mayan/apps/sources/tasks.py#L24-L33) | Acquire lock `task_source_process_document-<id>`; if held, **skip silently** | lock name string | overlapping ticks prevented; starvation invisible |
| 2 | backend | [watch_folder L88–96](../../../mayan/apps/sources/source_backends/watch_folder_backends.py#L88-L96) | `path.lstat()` + `is_dir()` sanity; regex include/exclude filters | server FS path | admin-controlled arbitrary path = trust boundary |
| 3 | backend | [watch_folder L100–112](../../../mayan/apps/sources/source_backends/watch_folder_backends.py#L100-L112) | First matching file → `SharedUploadedFile`, then **`entry.unlink()`** deletes the original, returns a 1-tuple | one file per tick | copy-then-delete is not atomic; crash between = file ingested? deleted? check both |
| 4 | mixin | [mixins L106–120](../../../mayan/apps/sources/source_backends/mixins.py#L106-L120) | For each staged file, dispatch `task_process_document_upload` with type/label/language/user IDs | primitive kwargs | |
| 5 | worker | [tasks L64–93](../../../mayan/apps/sources/tasks.py#L64-L93) | Re-fetch rows; `OperationalError` → retry with backoff | | |
| 6 | worker | [tasks L95–103](../../../mayan/apps/sources/tasks.py#L95-L103) | `source.handle_file_object_upload(...)` → document creation path; staging deleted | | |
| 7 | task wrapper | [tasks L39–53](../../../mayan/apps/sources/tasks.py#L39-L53) | Errors logged to `source.error_log`; success clears the log | error rows | *investigate:* if the `Source.objects.get` in step 1's `try` raised, `source` is unbound and the `except` block itself would `NameError` ([L36–L51](../../../mayan/apps/sources/tasks.py#L36-L51)) |

Validation and authorization: none per-file — the *source configuration* is the authorization (admin set it up). That is a real design position: machine channels inherit trust from their configurer.

Persistence and side effects: staging rows; **deletion of the watched file** (step 3); error-log rows on the source.

Tests that cover it: `mayan/apps/sources/tests/` contains per-backend tests (`test_watch_folder_source_backend*`).

What juniors usually miss: only **one file per tick** is ingested (the `return` inside the loop at [L112](../../../mayan/apps/sources/source_backends/watch_folder_backends.py#L100-L112)) — a folder of 10,000 files drains at one per interval. That's a throughput cliff hiding in a `return` statement.

What seniors notice: the lock has a timeout (`DEFAULT_SOURCES_LOCK_EXPIRE`) so a dead worker can't wedge the source forever; the silent skip on `LockError` means monitoring must watch *lag*, not errors. And the copy-then-delete gives at-least-once semantics — duplicate ingestion is possible; dedupe relies on the `duplicates` app after the fact.

Interview angle: "How would you build a folder-watcher that feeds a pipeline?" — locks, ticks, at-least-once, one-vs-many per tick. Card [Q5, 04-system-design](../08-interview-prep/04-system-design-from-this-repo.md).

Drill: propose the two-line change to ingest N files per tick and list what new risks appear. *Basic:* loop instead of return. *Solid:* bounded batch + lock duration risk. *Strong:* you tie batch size to lock timeout and worker memory, and mention fairness across sources.

---

## Flow 3: Authorization on read (`GET /api/v4/documents/<id>/`)

Why this flow matters: it is the security boundary. Every list/detail in UI and API funnels into one function.

Open these files first:
- [rest_api/generics.py](../../../mayan/apps/rest_api/generics.py#L120-L133) — RetrieveAPIView wiring
- [rest_api/filters.py](../../../mayan/apps/rest_api/filters.py#L6-L24) — the filter backend
- [acls/managers.py](../../../mayan/apps/acls/managers.py#L268-L294) — `restrict_queryset`
- [acls/managers.py](../../../mayan/apps/acls/managers.py#L31-L231) — `_get_acl_filters`

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | view | [document_api_views L23–L39](../../../mayan/apps/documents/api_views/document_api_views.py#L23-L39) | `queryset = Document.valid.all()`; per-method permission map | `mayan_object_permissions['GET']` | forgetting the map = open endpoint (list base class also demands `MayanPermission`) |
| 2 | filter backend | [filters L7–L16](../../../mayan/apps/rest_api/filters.py#L7-L16) | Looks up the permission for this HTTP method, calls `restrict_queryset` | queryset → queryset | runs on **every** request; cost matters |
| 3 | ACL manager | [managers L269–L277](../../../mayan/apps/acls/managers.py#L269-L277) | Anonymous → `none()`. Direct role permission / staff / superuser → whole queryset | short-circuit | staff bypass is broader than juniors expect ([user_has_this](../../../mayan/apps/permissions/models.py#L193-L216)) |
| 4 | ACL manager | [managers L278–L290](../../../mayan/apps/acls/managers.py#L278-L290) | Else build Q-filters: direct ACL rows + **inherited** ACLs (document_type → document), OR-reduced | `Q(id__in=<subquery>)` | recursion over `ModelPermission.get_inheritances` |
| 5 | DRF | | `get_object()` on the filtered queryset → 404 if excluded | | denial is 404: no resource-existence oracle |

Validation and authorization: this *is* the authorization. Two-layer model: role-level (`Permission.check_user_permissions`, [permissions/classes.py L65–L74](../../../mayan/apps/permissions/classes.py#L65-L74)) then object-level ACLs with inheritance.

Persistence and side effects: read-only; but note the events on detail views in UI (`document_viewed`) — reads can write audit rows.

Tests that cover it: the `no_permission` / `with_access` pairs, e.g. [test_document_api.py L138–L168](../../../mayan/apps/documents/tests/test_document_api.py#L138-L168).

What juniors usually miss: there is no `if not allowed: raise` anywhere near the view — absence of rows *is* denial. Grepping for "403" to find authz code fails here by design.

What seniors notice: `check_access` runs `restrict_queryset` per permission and ORs them ([managers L233–L266](../../../mayan/apps/acls/managers.py#L233-L266)) — correctness-first, N-subqueries cost. And case 2 of `_get_acl_filters` evaluates `if acl_filter:` ([L122](../../../mayan/apps/acls/managers.py#L119-L123)) — a truthiness check that **executes a query** to avoid an empty-Q AND bug; the comment explains the invariant. That's a masterclass in why `Q() & Q(...)` semantics matter.

Interview angle: "How would you implement per-object permissions?" — answer with filter-at-the-queryset, inheritance, 404-not-403. Cards [Q6–Q8, 03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: trace what happens for a user whose role has `document_view` granted **on the DocumentType** only. Walk the case numbers in `_get_acl_filters` that fire. *Basic:* access granted via inheritance. *Solid:* you name case 4 → case 2 recursion. *Strong:* you write the SQL shape (nested `IN` subqueries) and say where you'd look for an index if it's slow.

---

## Flow 4: `DocumentFile.save()` — the persistence pipeline

Why this flow matters: densest invariant cluster in the repo: hooks, signals, events, transaction, derived fields, and a cascade to the parent Document.

Open these files first:
- [document_file_models.py](../../../mayan/apps/documents/models/document_file_models.py#L428-L504) — `save()`
- [document_file_models.py](../../../mayan/apps/documents/models/document_file_models.py#L186-L213) — `checksum_update`
- [checkouts/apps.py](../../../mayan/apps/checkouts/apps.py#L66-L68) — a real pre-create hook registration

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | hooks | [L436–L444](../../../mayan/apps/documents/models/document_file_models.py#L436-L444) | `execute_pre_create_hooks` — checkouts can veto here | hook raising = upload blocked; ordering is registration `order` param |
| 2 | signal | [L449–L451](../../../mayan/apps/documents/models/document_file_models.py#L449-L451) | `signal_mayan_pre_save` (custom, carries `user`) | any listener can break saves |
| 3 | DB | [L453](../../../mayan/apps/documents/models/document_file_models.py#L453) | row insert (file already streamed to storage by `FileField`) | storage write happens **before/outside** the later transaction |
| 4 | event | [L460–L464](../../../mayan/apps/documents/models/document_file_models.py#L460-L464) | `event_document_file_created` committed | audit row even if step 5 fails? — check ordering |
| 5 | derived fields | [L466–L472](../../../mayan/apps/documents/models/document_file_models.py#L466-L472) | inside `transaction.atomic()`: checksum → mimetype → size → save → `page_count_update` | checksum streams whole file (block size setting at [L186–L197](../../../mayan/apps/documents/models/document_file_models.py#L186-L197)) |
| 6 | parent | [L479–L484](../../../mayan/apps/documents/models/document_file_models.py#L479-L484) | Document `is_stub=False`, label defaulted from filename | `_event_ignore` suppresses a duplicate edit event |
| 7 | signals | [L492–L500](../../../mayan/apps/documents/models/document_file_models.py#L492-L500) | `signal_post_document_file_upload`; first file also fires `signal_post_document_created` — OCR/parsing/indexing listen here | listener fan-out is the app's real "pipeline start" |

Validation and authorization: none — models trust their callers; boundaries did authz upstream. That's a documented layering decision you should be able to defend and attack.

Persistence and side effects: storage bytes (step 3), five field updates, page rows, events, downstream tasks via signals.

Tests that cover it: [test_document_file_models.py](../../../mayan/apps/documents/tests/test_document_file_models.py) (model-level), checkout blocking at [checkouts tests](../../../mayan/apps/checkouts/tests/test_models.py) (`test_blocking_new_files`, `test_file_creation_blocking`).

What juniors usually miss: `DocumentFile` streams to storage when Django saves the `FileField` — the "transaction" in step 5 protects derived *metadata*, not the bytes. A DB rollback there leaves an orphaned storage object.

What seniors notice: the delete path mirror: [`delete()` L220–L236](../../../mayan/apps/documents/models/document_file_models.py#L220-L236) removes pages, storage file, cache partition, then the row — and then flips `document.is_stub = False` when the file count hits zero ([L231–L234](../../../mayan/apps/documents/models/document_file_models.py#L231-L234)). *Investigate:* a document with zero files matches the textbook definition of a stub (`is_stub` help text at [document_models L98–L104](../../../mayan/apps/documents/models/document_models.py#L98-L104)); `False` here reads inverted. Fixing it (or explaining it) is Ticket 3 in [06-contribution-practice](../06-contribution-practice/01-good-first-tickets.md).

Interview angle: "What belongs inside a DB transaction?" — this flow is the perfect worked example of bytes-vs-metadata consistency. Card [Q9, 03-api-and-data-modeling](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: build the failure table — for a crash after each step 1–7, write down: DB state, storage state, user-visible state, and who cleans up. *Basic:* 4 rows right. *Solid:* all 7 with the reaper tasks named. *Strong:* you identify the two states nobody cleans up (orphaned storage blob; audit event for a file whose derivation failed) and propose the smallest fix.

---

## Flow 5: OCR pipeline (chord fan-out/fan-in)

Why this flow matters: the repo's most sophisticated Celery topology; teaches idempotency, chords, and error-log visibility.

Open these files first:
- [ocr/tasks.py](../../../mayan/apps/ocr/tasks.py#L17-L49) — coordinator
- [ocr/tasks.py](../../../mayan/apps/ocr/tasks.py#L51-L90) — per-page task
- [ocr/tasks.py](../../../mayan/apps/ocr/tasks.py#L92-L131) — finisher

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | signal listener | `ocr/handlers.py` (wired in ocr/apps.py) | new version → `task_document_version_ocr_process` | auto-OCR per document-type toggle |
| 2 | coordinator | [L31–L43](../../../mayan/apps/ocr/tasks.py#L31-L43) | Build one subtask **per page**, run as `chord(header)(finisher)` | chord requires a result backend (Redis — compose file) |
| 3 | per-page | [L74–L90](../../../mayan/apps/ocr/tasks.py#L74-L90) | Render page image (via cache), run OCR backend, store `DocumentVersionPageOCRContent`; retries on missing cache file, `LockError`, `OperationalError` | retry storms under load; content row upserted → idempotent |
| 4 | finisher | [L92–L117](../../../mayan/apps/ocr/tasks.py#L92-L117) | Clears the version error log, commits `event_ocr_document_version_finished` | runs only if **all** header tasks succeeded |
| 5 | errors | [L44–L48](../../../mayan/apps/ocr/tasks.py#L44-L48) | coordinator exceptions land in `document_version.error_log` | user-visible error surface — this is the observability pattern |

What juniors usually miss: `ignore_result=True` on the coordinator but **not** on the per-page task ([L51](../../../mayan/apps/ocr/tasks.py#L51-L53)) — chords need header results; "ignore results everywhere" would silently break the join.

What seniors notice: the finisher's `except OperationalError` block at [L126–L131](../../../mayan/apps/ocr/tasks.py#L118-L131) is **unreachable** — it follows `except Exception` which already catches it. Harmless here (both re-raise-ish) but a textbook exception-ordering bug; it's a good-first-ticket. Also: a single permanently-failing page means the finisher never runs and the error log is never cleared — partial-completion visibility relies on the per-page error rows.

Interview angle: fan-out/fan-in, "how do you know when N parallel jobs are done?" Card [Q7, 01-js-ts-node-deep-dive](../08-interview-prep/01-js-ts-node-deep-dive.md) (Promise.all analogy) and the debugging round in [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).

Drill: Promise.all vs chord — write 5 lines contrasting failure semantics (one rejection vs one task exhausting retries). *Strong:* you mention Promise.allSettled as the analog of collecting per-page error rows.

---

## Flow 6: Workflow transition

Why this flow matters: state machines are a top-3 system design interview topic, and this one is event-sourced.

Open these files first:
- [workflow_instance_models.py](../../../mayan/apps/document_states/models/workflow_instance_models.py#L82-L104) — `do_transition`
- [L132–L143](../../../mayan/apps/document_states/models/workflow_instance_models.py#L132-L143) — `get_current_state`
- [L171–L195](../../../mayan/apps/document_states/models/workflow_instance_models.py#L171-L195) — `get_transition_choices`
- [L282–L311](../../../mayan/apps/document_states/models/workflow_instance_models.py#L282-L311) — log entry `save()` runs state actions

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | view/API | document_states views | user picks a transition for a workflow instance | |
| 2 | model | [L86](../../../mayan/apps/document_states/models/workflow_instance_models.py#L86) | membership check: transition ∈ `get_transition_choices(_user)` | choices = origin-state transitions ∩ ACL ∩ condition templates |
| 3 | model | [L92–L99](../../../mayan/apps/document_states/models/workflow_instance_models.py#L92-L99) | append `WorkflowInstanceLogEntry` (comment, extra_data JSON) | **no lock**: two concurrent transitions both validate against the same "current" state |
| 4 | log save | [L291–L309](../../../mayan/apps/document_states/models/workflow_instance_models.py#L291-L309) | exit actions of origin state, then entry actions of destination state, run **synchronously** | slow/failing action = failed transition; actions can trigger events that trigger more actions ([get_context refresh comment L120–L124](../../../mayan/apps/document_states/models/workflow_instance_models.py#L120-L130)) |
| 5 | state | [L132–L143](../../../mayan/apps/document_states/models/workflow_instance_models.py#L132-L143) | "current state" is now the last entry's destination | state is *derived*, never stored — replayable history for free |

What juniors usually miss: there is no `current_state` column. UPDATE-less design: the log *is* the state. Reporting "what state is doc X in" costs a query over log entries.

What seniors notice: `do_transition` wraps everything in `try/except AttributeError` ([L101–L104](../../../mayan/apps/document_states/models/workflow_instance_models.py#L101-L104)) to paper over "no initial state," which also swallows any real `AttributeError` from deep inside state actions in production (`DEBUG` re-raises). And the concurrency gap in step 3 — two users can both take mutually-exclusive transitions; last write wins in *history order*.

Interview angle: "Design an approval workflow" — event-sourced state, ACL-per-transition, condition templates, escalation ([check_escalation L63–L80](../../../mayan/apps/document_states/models/workflow_instance_models.py#L63-L80)). This is the backbone of [08/04 system design](../08-interview-prep/04-system-design-from-this-repo.md) variation 2.

Drill: write the `select_for_update`-based fix for the double-transition race, then argue *against* shipping it. *Strong:* you weigh lock contention + action side effects inside a longer transaction vs the rarity/impact of the race, and propose a unique-constraint-shaped alternative (e.g. log entry carries expected predecessor).

---

## Flow 7: Search indexing on write

Why this flow matters: read-model maintenance via signals + queue — the "how does search stay fresh?" answer.

Open these files first:
- [dynamic_search/handlers.py](../../../mayan/apps/dynamic_search/handlers.py#L116-L125) — save handler
- [dynamic_search/tasks.py](../../../mayan/apps/dynamic_search/tasks.py#L41-L89) — index task
- [documents/search.py](../../../mayan/apps/documents/search.py#L22-L56) — what's searchable

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | signal | [handlers L116–L125](../../../mayan/apps/dynamic_search/handlers.py#L116-L125) | post-save → `task_index_instance.apply_async(app_label, model, pk)` | *synchronous DB write, async index write* → search lags writes (eventual consistency) |
| 2 | task | [tasks L60–L63](../../../mayan/apps/dynamic_search/tasks.py#L60-L63) | instance gone by the time worker runs → **retry** (`DoesNotExist` → `self.retry`) | retries a row that will never return if truly deleted |
| 3 | task | [tasks L65–L71](../../../mayan/apps/dynamic_search/tasks.py#L65-L71) | backend indexing; `LockError`/`DynamicSearchRetry` → retry w/ exponential backoff caps | |
| 4 | related | [handlers L56–L81](../../../mayan/apps/dynamic_search/handlers.py#L56-L81) | saving a *related* object (e.g. tag) re-indexes affected documents via reverse-path resolution | M2M changes get their own task ([handlers L84–L113](../../../mayan/apps/dynamic_search/handlers.py#L84-L113)) |
| 5 | delete | [handlers L13–L22](../../../mayan/apps/dynamic_search/handlers.py#L13-L22) + [tasks L27–L38](../../../mayan/apps/dynamic_search/tasks.py#L27-L38) | deindex task **re-fetches the instance from the DB** | *investigate:* if the row is already deleted when the worker runs, `.get()` raises unhandled `DoesNotExist` — stale entries may linger; check where the signal connects (pre vs post delete) |

What juniors usually miss: search results are a different consistency domain from list views. A test asserting "uploaded doc appears in search" must run the task eagerly or flush the queue.

What seniors notice: full reindex is chunked (`get_id_groups`) and fanned out ([tasks L153–L165](../../../mayan/apps/dynamic_search/tasks.py#L153-L165)) — bounded memory, parallel, resumable-ish. Compare with a naive `for doc in Document.objects.all()`.

Interview angle: "How do you keep a search index in sync with the DB?" — dual-write via queue, retries, eventual consistency, full-rebuild path. Card [Q10, 03-api-and-data-modeling](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: enumerate the three ways the index can go stale here (dropped task, deindex DoesNotExist path, backend LockError exhausting retries) and one detection mechanism for each.

---

## Flow 8: Trash → destruction lifecycle

Why this flow matters: soft delete with policy-driven hard delete; teaches reversibility and blast radius.

Open these files first:
- [document_models.py](../../../mayan/apps/documents/models/document_models.py#L142-L163) — the two-phase `delete()`
- [documents/tasks.py](../../../mayan/apps/documents/tasks.py#L299-L332) — trash-can emptying fan-out
- [documents/queues.py](../../../mayan/apps/documents/queues.py#L50-L69) — retention period checks on schedule

| Step | Owner | What happens | Risk |
| --- | --- | --- | --- |
| 1 | `Document.delete()` first call | flips `in_trash`, stamps time, commits `event_document_trashed` — files untouched | reversible; UI "delete" is this |
| 2 | `Document.valid` manager | trashed docs vanish from all normal lists/APIs while `Document.objects` still sees them | queryset choice = access policy |
| 3 | periodic checks | per-type `trash_time_period` / `delete_time_period` enforce retention ([queues L50–L69](../../../mayan/apps/documents/queues.py#L50-L69)) | policy lives on DocumentType — type change moves the goalposts (ties to Flow 1/change-type) |
| 4 | second `delete()` | cascades: each `DocumentFile.delete()` removes storage bytes + cache partitions, then the row ([document_file_models L220–L236](../../../mayan/apps/documents/models/document_file_models.py#L220-L236)) | irreversible; storage delete before row delete → crash leaves row pointing at nothing (benign?) verify |
| 5 | trash-can empty | one task **per document** ([tasks L299–L311](../../../mayan/apps/documents/tasks.py#L299-L311)) | failure isolation: one poisoned doc doesn't stop the sweep |

What juniors usually miss: `delete(to_trash=False)` exists for true hard delete ([document_models L142–L146](../../../mayan/apps/documents/models/document_models.py#L142-L146)) and is used in the upload-failure cleanup ([tasks L172–L180](../../../mayan/apps/documents/tasks.py#L172-L180)).

What seniors notice: the event actors: trashing targets the document, destruction targets the **document type** ([L161–L163](../../../mayan/apps/documents/models/document_models.py#L154-L163)) — because after destruction the document can't be referenced. Audit design under cascade is a subtle craft.

Interview angle: "How do you implement soft delete without littering every query with `WHERE deleted=false`?" — dedicated managers + proxy models. Card [Q11, 03-api-and-data-modeling](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: find every manager on `Document` and state each one's contract ([document_models L106–L108](../../../mayan/apps/documents/models/document_models.py#L106-L108) plus [managers.py](../../../mayan/apps/documents/managers.py)). *Strong:* you can say which views intentionally use `objects` vs `valid` vs `trash`, and what breaks if someone "fixes" a `valid` to `objects`.
