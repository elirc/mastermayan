# System design from this repo

Reverse-engineer Mayan into a whiteboard exercise. Prompt: **"Design a document management system: users upload files, the system extracts text so they're searchable, organizes them, controls who can see what, and keeps an audit trail."** Below is the walkthrough with, at each step, what Mayan actually chose (anchored) and a stronger/simpler alternative — plus what junior/mid/senior answers sound like. Then four variation prompts.

## Step 0: Clarify requirements (always start here)
Ask: scale (docs, users, upload rate)? multi-tenant? file types/sizes? search latency tolerance? retention/compliance? on-prem or cloud? A senior spends the first 3 minutes here; a junior starts drawing boxes. Mayan's answers: self-hosted, single-tenant-ish (org scoping), any file type, OCR-heavy, compliance-oriented (audit + retention).

## Step 1: Core entities
Mayan: **Document** (logical) → **DocumentFile** (immutable bytes) → **DocumentVersion** (page mapping) → pages; plus Type, Tag/Cabinet/Metadata, Workflow, ACL, Event ([data model](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
- *Junior:* one `documents` table with a `file` column and a `text` column.
- *Mid:* separates file bytes from metadata; adds a types/tags layer.
- *Senior:* immutable files + mutable version-as-page-mapping for audit history and cheap reordering ([document_version_models.py L315–L338](../../../mayan/apps/documents/models/document_version_models.py#L315-L338)); notes it's more complex than most products need and would justify it by the compliance requirement.
**Stronger/simpler alternative:** if history isn't required, collapse file+version — Mayan's split is *earned* by audit needs, not free.

## Step 2: Upload API
Mayan: stage bytes → return early → worker processes → cleanup ([Flow 1](../01-codebase-cartography/05-key-flows.md)).
- *Junior:* synchronous save in the request.
- *Mid:* async with a staging table and ID-passing; poll for readiness.
- *Senior:* adds idempotency key (Mayan lacks one — a real gap), resumable uploads for large files, and names the consistency boundary (stub until worker done).

## Step 3: Text extraction (OCR) pipeline
Mayan: per-page tasks fanned out via chord, finisher on completion, per-version error log ([ocr/tasks.py L17–L48](../../../mayan/apps/ocr/tasks.py#L17-L49)).
- *Junior:* OCR inline after upload.
- *Mid:* queue it; process pages in parallel; store extracted text.
- *Senior:* fan-out/fan-in with completion semantics; failure isolation (one bad page); the chord-never-completes failure mode and how to detect it; a dedicated latency-insensitive worker tier so OCR can't starve interactive work.

## Step 4: Search
Mayan: pluggable backend (Whoosh/Elasticsearch), index updated async on write via signals, chunked full rebuild ([Flow 7](../01-codebase-cartography/05-key-flows.md)).
- *Junior:* `WHERE text LIKE '%q%'`.
- *Mid:* a real search engine, indexed asynchronously; accept eventual consistency.
- *Senior:* dual-write via queue with retry; the three staleness sources + rebuild path; when Postgres FTS suffices vs when you need Elasticsearch (scale + relevance).

## Step 5: Authorization
Mayan: roles (via groups) + per-object ACLs with inheritance, enforced as a queryset filter, 404 on denial ([authz](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)).
- *Junior:* an `owner_id` column + `if owner == user`.
- *Mid:* RBAC + per-object checks; filter at the query.
- *Senior:* inheritance so grants scale by container; ReBAC-lite; the reasoning-cost tradeoff; the bypass-population caveat. This is Mayan's strongest area — spend time here.

## Step 6: Audit trail
Mayan: every mutation emits an event via a method decorator, with actor/target/action ([events](../03-architecture-and-patterns/05-pattern-catalog.md) pattern 5).
- *Mid:* an events table written by the service layer.
- *Senior:* enforce at the model so nothing can forget; the synchronous-fan-out cost and how you'd make notifications async while keeping the audit write atomic ([kata 3](../06-contribution-practice/04-refactor-and-design-katas.md)).

## Step 7: Scaling concerns (name these unprompted)
- Web/worker split so slow work never blocks requests ([worker tiers](../01-codebase-cartography/04-runtime-and-tooling-map.md)).
- The ACL query is the read-path hotspot at scale ([risk 1](../03-architecture-and-patterns/06-architecture-critique.md)) — measure, index, maybe materialize.
- Storage is pluggable (local→S3); DB is the bottleneck for metadata; search scales independently.
- Blast radius via queue isolation; backpressure via bounded queues; reapers for crash residue.

## Answer-quality ladder (memorize the shape)
At every step: junior gives *one* option; mid gives an option **with a tradeoff**; senior gives the option, the tradeoff, the **failure mode**, and **what they'd measure before optimizing**. Interviewers score the ladder, not the box diagram.

---

## Variation 1: "Now make it multi-tenant (strict isolation)"
Discuss: tenant scoping on every query (Mayan has org scoping but the ACL layer is the real isolation); the danger of a new view forgetting the filter (structural fix: base classes enforce it); per-tenant storage prefixes; noisy-neighbor on shared workers (per-tenant queues or quotas — Mayan has a `quotas` app). Senior point: isolation must be *default-on and hard to bypass*, which is exactly why Mayan filters at the queryset in a base class.

## Variation 2: "Add real-time collaboration / live status"
Mayan is request-response with no push. Discuss: WebSockets/SSE for OCR-progress and workflow updates; the challenge of pushing from *workers* (they'd publish to a channel layer); why the current honest-pending model (poll the stub) is the simpler 80% solution. Senior point: don't add a stateful realtime layer unless the product needs it — for a document vault, progress polling is often enough.

## Variation 3: "10x the traffic"
Where it breaks first (hypotheses, stated as such): ACL subqueries on list views; the DB as the metadata bottleneck; OCR worker saturation; cache lock contention ([scenario 4](../05-quality-engineering/03-systematic-debugging.md)). Fixes in order: measure → index/cache the authz path → scale worker tiers independently → read replicas for list/search → shard by tenant if needed. Senior point: scale the *proven* bottleneck; Mayan's architecture already isolates the pieces so you can scale them separately.

## Variation 4: "Add an approval workflow with SLAs"
Mayan already has this — event-sourced state, ACL-per-transition, escalations ([Flow 6](../01-codebase-cartography/05-key-flows.md)). Discuss: state as last-log-entry (replayable, auditable) vs a state column (simpler, no history); the transition race and its fix; escalation via periodic checks vs scheduled jobs; running state-actions inline vs async (Mayan runs them inline — a latency/failure-coupling risk). Senior point: event-sourcing is *earned* by the audit/escalation requirement; for a simple two-state flow a boolean is fine.

**Drill:** do the full walkthrough on a whiteboard in 35 minutes, out loud, citing at least six anchors from memory. Then do variation 1 in 10 minutes. Record it. *Strong:* at every step you volunteer the failure mode and the "measure first" without being prompted, and you can say where Mayan is over-engineered for a simpler product (files/versions split, event-sourcing) — knowing when the fancy choice is *wrong* is the top signal.
