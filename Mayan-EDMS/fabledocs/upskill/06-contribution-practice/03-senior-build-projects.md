# Senior build projects

Six projects a maintainer might realistically accept. Each has the full senior shape: problem, value, design checklist, decisions, likely files, migration, tests, security, performance, rollout/rollback, open questions, stretch. Do them as design documents even if you can't merge — the artifact *is* the skill.

---

## Project 1: Task observability subsystem (dead-letter + metrics)

**Duration:** ~2 weeks. **Problem:** exhausted/failed background tasks vanish into logs; operators have no failure rate, no retry, no dead-letter ([05/06 gaps](../05-quality-engineering/06-observability-and-operations.md)).
**Product value:** turns "my upload never finished" from a support ticket into a dashboard.
**Design checklist:** Celery `task_failure`/`task_retry` signals vs explicit calls; a `FailedTask` model (task name, kwargs, exception, retries, first/last seen); admin list + manual re-dispatch; a metrics surface (`mayan_statistics` integration).
**Architecture decisions:** capture at the Celery signal layer (one integration point, catches all tasks) vs per-task (surgical but incomplete); **decision:** signal layer, opt-out per task. Store kwargs redacted (PII).
**Likely files:** new `task_manager` submodule; wiring in [task_manager/apps.py](../../../mayan/apps/task_manager/apps.py); admin view.
**Migration:** one additive model. **Tests:** simulate a task exhausting retries; assert a `FailedTask` row + re-dispatch works. **Security:** redact kwargs; the view is a maintenance permission ([ViewPermissionCheckViewMixin](../../../mayan/apps/views/mixins.py#L629-L649)). **Performance:** failure path only — negligible. **Rollout/rollback:** additive, feature-flag the signal handler. **Open questions:** retention policy; does re-dispatch replay side effects safely (idempotency per task)? **Stretch:** alerting thresholds.
**Interview story potential:** "I built the missing failure-visibility layer for a queue-based system."

## Project 2: ACL performance hardening

**~3 weeks.** **Problem:** list-view authorization builds recursive subqueries; suspected to degrade at ~10⁶ docs (hypothesis — [03/06 risk 1](../03-architecture-and-patterns/06-architecture-critique.md)).
**Value:** keeps large installs usable.
**Checklist:** benchmark harness with seeded volume; `EXPLAIN ANALYZE` the ACL queryset; try (a) composite index on the ACL table, (b) caching the role-wide short-circuit, (c) a materialized effective-permission table refreshed on grant/revoke.
**Decisions:** measure before building; prefer (a)+(b) (low risk) before (c) (big RFC, cache-invalidation problem).
**Likely files:** [acls/managers.py](../../../mayan/apps/acls/managers.py#L31-L294), a new benchmark test, possibly a migration for the index.
**Migration:** index add is online-safe in Postgres (`CREATE INDEX CONCURRENTLY` — but Django migrations need `atomic=False`). **Tests:** `assertNumQueries` + timing regression. **Security:** correctness is paramount — a caching bug is a permission bug; TTL-free invalidation only. **Performance:** the whole point; report before/after. **Rollback:** (a)(b) trivially reversible; (c) needs dual-read. **Open questions:** does materialization break inheritance semantics? **Stretch:** ReBAC-style precomputation.
**Story potential:** "I profiled and de-risked the authorization hot path."

## Project 3: Structured, correlated logging + request/task tracing

**~2 weeks.** **Problem:** debugging async flows means grepping unstructured logs across processes with no correlation ID ([05/03 debugging](../05-quality-engineering/03-systematic-debugging.md)).
**Value:** cuts async incident time-to-root-cause dramatically.
**Checklist:** inject a correlation ID at the web edge; propagate it into task kwargs; structured (JSON) log formatter in the `logging` app; a log line at each Flow boundary.
**Decisions:** propagate via an extra task kwarg (explicit, survives the queue) vs context vars (lost across the broker) — **decision:** kwarg, defaulted, additive to task signatures (respect the queue contract, [02/03 case study 3](../02-stack-and-language-mastery/03-type-system-and-contracts.md)).
**Likely files:** middleware in [views/middleware/](../../../mayan/apps/views/middleware/); task decorators; logging config.
**Migration:** none. **Tests:** assert the ID appears in a task's log context. **Security:** don't log bodies/secrets. **Performance:** log volume. **Rollback:** formatter revert. **Open questions:** integrate OpenTelemetry or stay stdlib? **Stretch:** span export.
**Story potential:** "made a distributed system debuggable."

## Project 4: Pluggable virus/content scanning on upload

**~3 weeks.** **Problem:** untrusted files are stored and converted with no content scanning ([05/05 uploads row](../05-quality-engineering/05-security-checklist.md)).
**Value:** table-stakes for regulated deployments.
**Checklist:** a scanning backend interface (mirror the engine-backend pattern, [pattern 9](../03-architecture-and-patterns/05-pattern-catalog.md)); a pre-create hook that scans the staged file ([DocumentFile pre-create hooks](../../../mayan/apps/documents/models/document_file_models.py#L135-L155), same slot checkouts use); quarantine on hit.
**Decisions:** scan the `SharedUploadedFile` before `file_new` (fail fast, no orphan storage) vs post-save (simpler, worse); **decision:** pre-create hook + `Warning`-style veto ([existing pattern, documents/tasks.py L90–L96](../../../mayan/apps/documents/tasks.py#L82-L96)).
**Likely files:** new app or `sources`/`documents` hook; backend for ClamAV. **Migration:** quarantine model. **Tests:** clean file passes, EICAR test string quarantines. **Security:** the feature *is* security; sandbox the scanner. **Performance:** adds latency to the upload task — worker tier placement matters. **Rollback:** backend `none` disables. **Open questions:** scan versions on remap? **Stretch:** async scan with pending state.
**Story potential:** "designed a pluggable security scanning layer following the codebase's own backend pattern."

## Project 5: Bulk-safe document operations API

**~2 weeks.** **Problem:** bulk actions (trash N, retag N) either loop synchronously or aren't exposed; batch endpoints exist ([rest_api batch_requests](../../../mayan/apps/rest_api/urls.py#L20-L24)) but per-item failure semantics are unclear.
**Value:** power users manage thousands of docs.
**Checklist:** a bulk endpoint that ACL-filters the id set, fans out per-item tasks (like trash-empty, [documents/tasks.py L299–L311](../../../mayan/apps/documents/tasks.py#L299-L311)), returns a job handle; per-item results.
**Decisions:** all-or-nothing vs best-effort-with-report — **decision:** best-effort + result rows (matches at-least-once philosophy).
**Likely files:** new API view; a job/result model. **Migration:** result model. **Tests:** partial failure returns partial success; ACL filters the set before dispatch (no privilege bypass). **Security:** filter the id set through `restrict_queryset` — the classic bulk-IDOR trap. **Performance:** fan-out bounded. **Rollback:** endpoint removal. **Open questions:** progress reporting UX. **Stretch:** cancellation.
**Story potential:** "designed bulk operations without a bulk-IDOR hole."

## Project 6: Type-annotation pilot at the task + serializer boundaries

**~2 weeks.** **Problem:** no static typing; the highest-risk cross-boundary code (task kwargs, serializer create/update) has zero compile-time checks ([02/03](../02-stack-and-language-mastery/03-type-system-and-contracts.md)).
**Value:** catches the `ignore_results`-class bugs before prod; documents contracts.
**Checklist:** add annotations + `mypy` (permissive config) to one app's `tasks.py` and `serializers/`; wire an optional CI job; document the incremental-adoption plan.
**Decisions:** which app first (documents — highest traffic, best-tested); strictness (start permissive). **Likely files:** one app's tasks/serializers; a `mypy.ini`; CI stage. **Migration:** none. **Tests:** mypy passes in CI; behavior unchanged. **Security:** none direct. **Performance:** CI time. **Rollback:** drop the CI stage. **Open questions:** will maintainers accept a typing direction at all? (**This is the real risk — socialize first.**) **Stretch:** typed settings.
**Story potential:** "introduced incremental typing to a large untyped codebase without a big-bang rewrite."

---

**Choosing:** Projects 1 and 3 give the best value-to-risk and produce the strongest STAR stories for a mid-level loop. Project 2 is the most impressive *if* you have a real perf problem to point at. Project 6 is a political skill exercise as much as technical — attempt the design note even if you never merge it. For any project, the first deliverable is the RFC ([07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)), and the interview version is a system-design walkthrough ([08/04](../08-interview-prep/04-system-design-from-this-repo.md)).
