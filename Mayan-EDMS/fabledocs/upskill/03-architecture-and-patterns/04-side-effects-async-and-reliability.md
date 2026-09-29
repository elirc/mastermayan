# Side effects, async, and reliability

## The side-effect map

| Side effect | Triggered by | Executed | Anchor | Failure visibility |
| --- | --- | --- | --- | --- |
| File bytes → storage | `DocumentFile.save()` via FileField | sync, in-request/worker | [document_file_models.py L428–L453](../../../mayan/apps/documents/models/document_file_models.py#L428-L453) | exception to caller |
| Checksum/MIME/pages | new file | worker (same task) | [L466–L472](../../../mayan/apps/documents/models/document_file_models.py#L466-L472) | task retry/error log |
| OCR / parsing / indexing | `signal_post_document_file_upload` | queued tasks | [L492–L500](../../../mayan/apps/documents/models/document_file_models.py#L492-L500) | per-version `error_log` rows ([ocr/tasks.py L44–L48](../../../mayan/apps/ocr/tasks.py#L44-L48)) |
| Search index update | post-save/delete signals | queued w/ backoff | [dynamic_search/handlers.py L116–L125](../../../mayan/apps/dynamic_search/handlers.py#L116-L125) | logs only — silent staleness |
| Audit event + notifications | `@method_event` on save/delete | **sync, inline** | [events/classes.py L359–L429](../../../mayan/apps/events/classes.py#L359-L429) | exception propagates into the save |
| Workflow state actions (HTTP calls, signing, type changes…) | transition log entry `save()` | **sync, inline** | [workflow_instance_models.py L291–L309](../../../mayan/apps/document_states/models/workflow_instance_models.py#L282-L311) | transition fails with the action |
| Watched-folder file deletion | periodic source scan | worker | [watch_folder_backends.py L100–L112](../../../mayan/apps/sources/source_backends/watch_folder_backends.py#L100-L112) | source `error_log` |
| Email (mailer app), webhooks (workflow HTTP action) | user/workflow | worker/inline | mailer app; document_states workflow_actions | error logs |
| Cache prune/purge | cache writes, size change | inline under lock | [file_caching/models.py L105–L152](../../../mayan/apps/file_caching/models.py#L105-L152) | `FileCachingException` |

Two rows deserve your suspicion (flagged, not asserted): **events are synchronous inside `save()`** — the audit write, subscription fan-out, and notification inserts run per-save ([events/classes.py L387–L427](../../../mayan/apps/events/classes.py#L387-L429)); a notification-table problem becomes a document-save problem. And **workflow actions run inside the transition** — a slow webhook makes a user's POST hang; a crash leaves exit actions run but entry actions not (no compensation).

## Idempotency (the interview word, applied)

- **Retryable by construction:** per-page OCR upserts content keyed by page ([ocr/tasks.py L74–L84](../../../mayan/apps/ocr/tasks.py#L74-L90)); search indexing rewrites the whole doc entry; `F()`-based hit counting tolerates replays *of the increment* but not exactly-once semantics.
- **Not idempotent:** `task_document_upload` — replay = duplicate document (no dedupe key; Flow 1); watch-folder ingestion between copy and unlink (at-least-once by design).
- The repo's implicit stance: **at-least-once delivery + idempotent-or-reapable effects**. Reapers: stub cleanup ([documents/tasks.py L129–L137](../../../mayan/apps/documents/tasks.py#L129-L137)), staged-file deletion in every task exit path ([L103–L124](../../../mayan/apps/documents/tasks.py#L103-L124)), cache prune ([file_caching/models.py L105–L152](../../../mayan/apps/file_caching/models.py#L105-L152)).

## Retry policy taxonomy (steal this table)

| Pattern | Where | Why it's right |
| --- | --- | --- |
| Retry only `OperationalError` | upload tasks ([documents/tasks.py L37–L45](../../../mayan/apps/documents/tasks.py#L37-L45)) | transient DB blips heal; logic errors shouldn't loop |
| Retry `DoesNotExist` | indexer waiting for a row the signal beat to the queue ([dynamic_search/tasks.py L60–L63](../../../mayan/apps/dynamic_search/tasks.py#L60-L63)) | *race-aware* retry — the row will appear (commit lag) |
| Retry `LockError` + backoff caps | index/deindex ([dynamic_search/tasks.py L22–L26](../../../mayan/apps/dynamic_search/tasks.py#L22-L26)) | contention is transient; caps prevent storms |
| No retry, log to entity error log | source processing ([sources/tasks.py L39–L53](../../../mayan/apps/sources/tasks.py#L39-L53)) | humans must fix a bad folder path; retrying can't |

The missing quadrant repo-wide: **dead-letter handling**. A task that exhausts retries vanishes into logs. Proposing a lightweight failed-task table is senior project 4 in [06/03](../06-contribution-practice/03-senior-build-projects.md).

## Locks: the three backends and their honesty

`LockingBackend.get_backend()` resolves file-lock (single host), model-lock (DB table), or Redis lock — production compose picks Redis ([docker-compose.yml L12–L13](../../../docker/docker-compose.yml#L3-L16)). Read [file_lock.py](../../../mayan/apps/lock_manager/backends/file_lock.py#L62-L116) as a teaching object: JSON dict in a file, expiration timestamps, UUID ownership so you only release your own lock ([L103–L108](../../../mayan/apps/lock_manager/backends/file_lock.py#L94-L116)). Sharp edges worth an "investigate": `_release` catches `EOFError` but empty-file `json.loads('')` raises `JSONDecodeError` ([L98–L101](../../../mayan/apps/lock_manager/backends/file_lock.py#L94-L101)); and the module-level `threading.Lock` is `.release()`d at the end of the happy path only — an exception mid-block deadlocks the process's other threads ([L67–L92](../../../mayan/apps/lock_manager/backends/file_lock.py#L62-L92)). Every lock here has a **timeout** — the design accepts stale-lock takeover over permanent wedging ([L78–L85](../../../mayan/apps/lock_manager/backends/file_lock.py#L78-L92)); that means critical sections must tolerate a rare second entrant. Say *that* in an interview and you understand distributed locks better than most.

## Backpressure and overload

Queues buffer bursts; worker tiers keep heavy work off interactive lanes ([01/04 runtime map](../01-codebase-cartography/04-runtime-and-tooling-map.md)); `retry_backoff=True` spreads thundering herds ([sources/tasks.py L58](../../../mayan/apps/sources/tasks.py#L58-L63)); per-child task/memory caps recycle leaky workers ([workers.py L12–L36](../../../mayan/apps/task_manager/workers.py#L12-L36)). Missing: rate limiting at the API edge and queue-depth alerting (observability module picks this up).

**Drill (failure injection on paper):** for Flow 1, list every point where the process can die, and classify the residue: (a) self-healing, (b) reaped later, (c) permanent garbage, (d) user-visible inconsistency. *Basic:* 4 points. *Solid:* 8+, correctly classified. *Strong:* you identify the storage-blob-without-row case as (c), the stub-without-file as (b), and design the one metric that would expose class (c) growth (storage object count vs `DocumentFile.count()`).

**Interview angle:** idempotency, at-least-once vs exactly-once, retry taxonomies, and lock timeouts are the four async questions mid-level candidates fumble. You now have a named repo example for each. Cards: [08/01 Q7](../08-interview-prep/01-js-ts-node-deep-dive.md), [08/03 Q10](../08-interview-prep/03-api-and-data-modeling-questions.md), [08/04 variations](../08-interview-prep/04-system-design-from-this-repo.md).
