# API and data modeling — question cards

18 cards, the densest anchor set in the curriculum. This is where Mayan is strongest as an interview lab.

---

## Q1: Design a file-upload API endpoint
**Testing:** async design, large payloads, failure handling.
**Anchor:** [Flow 1](../01-codebase-cartography/05-key-flows.md), [document_serializers.py L74–L90](../../../mayan/apps/documents/serializers/document_serializers.py#L74-L90), [documents/tasks.py L52–L124](../../../mayan/apps/documents/tasks.py#L52-L124).
**Junior:** "POST the file, save it, return 200."
**Mid adds:** stage the bytes, return early (202-style), process in a worker; pass an *ID* not bytes to the queue; clean up staging on success and failure.
**Senior adds:** no idempotency key = duplicate risk on client retry; client must poll for readiness (stub → ready); checksum/dedupe happens in the worker so it can't be a synchronous response.
**Follow-up:** add resumable uploads; add an idempotency key.

## Q2: REST vs RPC-style action endpoints
**Anchor:** Mayan mixes both — RESTful resources plus action endpoints like `document-change-type` ([document_api_views.py L95–L110](../../../mayan/apps/documents/api_views/document_api_views.py#L95-L111), `ObjectActionAPIView` at [rest_api/generics.py L75–L117](../../../mayan/apps/rest_api/generics.py#L75-L117)).
**Mid adds:** pure REST struggles with verbs ("change type," "check out"); a POST action endpoint is pragmatic. Mayan uses resources for CRUD and actions for state transitions.
**Senior adds:** the consistency cost of mixed styles; how to keep actions discoverable (hyperlinked action URLs in the serializer, [L28–L31](../../../mayan/apps/documents/serializers/document_serializers.py#L23-L41)).

## Q3: API versioning
**Anchor:** `/api/v4/` in the URL ([rest_api/urls.py L27–L34](../../../mayan/apps/rest_api/urls.py#L27-L34)).
**Mid adds:** URL versioning is explicit and cache-friendly; alternatives are header/accept versioning. The point is a stable contract for existing clients.
**Senior adds:** the *hidden* version — Celery task kwargs are an unversioned contract across deploys ([02/03](../02-stack-and-language-mastery/03-type-system-and-contracts.md)); versioning the HTTP API doesn't protect in-flight messages.

## Q4: Pagination strategies
**Anchor:** DRF pagination on list views ([rest_api/generics.py L40–L56](../../../mayan/apps/rest_api/generics.py#L40-L56)); sorting via `_ordering` ([rest_api/filters.py L27–L31](../../../mayan/apps/rest_api/filters.py#L26-L31)).
**Mid adds:** offset/limit is simple but drifts under concurrent inserts and is slow deep in the list; cursor/keyset pagination is stable and fast but can't jump to page N.
**Senior adds:** which to pick for an audit log (cursor — append-heavy, deep scroll) vs a document list (offset acceptable). Failure mode: offset pagination + a hot-inserting table = duplicates/skips.

## Q5: Filtering and the N+1 trap in list responses
**Anchor:** serializer nested fields causing per-row queries (`file_latest`, [document_models.py L185–L187](../../../mayan/apps/documents/models/document_models.py#L185-L187)).
**Mid adds:** each nested serializer/method field can fire a query per row; fix with `select_related`/`prefetch_related`/annotate. Mayan has a `ModelQueryFields` hook for this.
**Senior adds:** prove it with `assertNumQueries`; the count must not scale with row count ([05/04](../05-quality-engineering/04-performance-thinking.md)).

## Q6: How do you implement per-object authorization (prevent IDOR)?
**Testing:** the headline security question.
**Anchor:** [restrict_queryset, acls/managers.py L268–L294](../../../mayan/apps/acls/managers.py#L268-L294); filter backend [rest_api/filters.py L6–L24](../../../mayan/apps/rest_api/filters.py#L6-L24).
**Junior:** "check if the user owns it in the handler."
**Mid adds:** filter at the *queryset* so forbidden rows never load; denial = 404 (no existence oracle); a forgotten check can't leak because the base class applies the filter.
**Senior adds:** inheritance so you grant on containers not items; the bypass populations (staff/superuser) must be intentional; test the denial per endpoint (the house pair convention).
**Follow-up:** multi-tenant isolation at 10x scale (variation in [04](04-system-design-from-this-repo.md)).

## Q7: Role-based vs relationship-based access control
**Anchor:** roles ([permissions/models.py L193–L216](../../../mayan/apps/permissions/models.py#L193-L216)) + object ACLs with inheritance ([acls/managers.py L125–L194](../../../mayan/apps/acls/managers.py#L125-L194)).
**Mid adds:** RBAC = user→role→permission (Mayan's tier 1, via group membership); ReBAC = permissions flow along relationships (Mayan's inheritance is ReBAC-lite: type→document).
**Senior adds:** the reasoning cost — "why can this user see this?" needs a tool ([get_inherited_permissions](../../../mayan/apps/acls/managers.py#L296-L310)); Google Zanzibar is the industrial ReBAC; Mayan's version is a pragmatic middle.

## Q8: Where should authorization live — and what's a subtle authz bug?
**Anchor:** the change-type asymmetry ([03/03 §edge cases](../03-architecture-and-patterns/03-validation-auth-and-permissions.md); test at [test_document_api.py L85–L112](../../../mayan/apps/documents/tests/test_document_api.py#L85-L112)).
**Mid adds:** authz belongs at the boundary (view/queryset), integrity at the model. The subtle bug: creating a document needs create-permission on the type, but *moving* one needs only edit-on-the-doc — so edit rights let you relocate into any type.
**Senior adds:** subtle authz bugs live at *related-object writes*; the fix changes a tested contract, so it's a rollout problem too (setting-gated, [06/M2](../06-contribution-practice/02-mid-level-feature-tickets.md)).
**This is a top-tier answer** — a real, evidenced, non-obvious authz finding. Rehearse it cold.

## Q9: What belongs inside a database transaction?
**Anchor:** the derived-field cluster in `DocumentFile.save` ([document_file_models.py L466–L484](../../../mayan/apps/documents/models/document_file_models.py#L466-L484)).
**Mid adds:** operations that must be all-or-nothing (checksum+mime+size+pages+parent-flag); keep external/slow work out (storage bytes, HTTP, queue sends).
**Senior adds:** Mayan's storage write is *outside* the transaction, so a rollback strands a blob (consistency boundary = DB is truth, storage is eventually reconciled); putting a webhook inside a transaction (workflow actions!) holds locks across the network — an anti-pattern.

## Q10: Keep a search index consistent with the database
**Anchor:** [Flow 7](../01-codebase-cartography/05-key-flows.md); [dynamic_search/handlers.py L116–L125](../../../mayan/apps/dynamic_search/handlers.py#L116-L125).
**Mid adds:** dual-write via a queue — commit to DB, enqueue an index update; accept eventual consistency; handle the row-not-yet-committed race with retry.
**Senior adds:** three staleness sources (dropped task, deindex on a deleted row, retry exhaustion) each need detection; a periodic full rebuild ([tasks L153–L165](../../../mayan/apps/dynamic_search/tasks.py#L153-L165)) is the repair path; the CDC/outbox alternative.

## Q11: Soft delete without WHERE-clause litter
**Anchor:** manager taxonomy + proxy ([document_models.py L106–L108, L142–L163](../../../mayan/apps/documents/models/document_models.py#L106-L163)).
**Mid adds:** a flag + a default manager that filters it; a separate manager for the trash view; proxy models for trash-specific behavior.
**Senior adds:** uniqueness constraints must account for soft-deleted rows; the wrong manager in one new query reintroduces the leak; retention policy drives hard delete ([Flow 8](../01-codebase-cartography/05-key-flows.md)).

## Q12: Schema migration safely, zero downtime
**Anchor:** [02-data-model §schema change](../03-architecture-and-patterns/02-data-model-and-persistence.md); `MayanMigratorTestCase`.
**Mid adds:** additive first (nullable/defaulted), backfill, then enforce; renames/drops are two-release; test migrations.
**Senior adds:** in-flight queue messages must remain valid across the deploy window (not just old *code*); reverse functions for `RunPython` or you can't roll back.

## Q13: Choosing a primary/identity key
**Anchor:** UUID on Document ([L53–L58](../../../mayan/apps/documents/models/document_models.py#L53-L58)) + `natural_key` ([L276–L278](../../../mayan/apps/documents/models/document_models.py#L276-L278)).
**Mid adds:** surrogate (autoincrement) for FK graphs; UUID for merge/exposure safety; natural key for cross-DB portability. Mayan uses all three intentionally.
**Senior adds:** UUID index locality cost; exposing sequential IDs leaks volume (Mayan exposes both pk and uuid — discuss).

## Q14: Modeling versioned/immutable data
**Anchor:** files immutable, versions as page mappings ([document_version_models.py L315–L338](../../../mayan/apps/documents/models/document_version_models.py#L315-L338)).
**Mid adds:** append-only facts + a mutable view layer gives history + cheap undo without copying bytes.
**Senior adds:** the "≤1 active version" invariant enforced by transaction convention not a constraint ([kata 7](../06-contribution-practice/04-refactor-and-design-katas.md)) — a place a partial unique index would harden.

## Q15: Concurrency control — optimistic vs pessimistic vs constraints
**Anchor:** checkout OneToOne backstop ([checkouts/models.py L100–L116](../../../mayan/apps/checkouts/models.py#L100-L116)), F() counters ([file_caching/models.py L415](../../../mayan/apps/file_caching/models.py#L406-L416)), workflow race ([Flow 6](../01-codebase-cartography/05-key-flows.md)).
**Mid adds:** the three tools — DB constraints (cheapest, Mayan's checkout), `F()`/atomic updates (lock-free counters), locks/`select_for_update` (expensive). Check-then-act races unless the DB backstops it.
**Senior adds:** the workflow transition has *no* backstop → a real race; the fix (expected-predecessor + unique constraint) avoids holding a lock across side effects.

## Q16: How do you test an API's authorization?
**Anchor:** the pair convention ([test_document_api.py L27–L112](../../../mayan/apps/documents/tests/test_document_api.py#L27-L112)).
**Mid adds:** per endpoint, `no_permission` → 404 + no state change + no events, and `with_access` → 2xx + exact events; a second-object test for IDOR.
**Senior adds:** the tests *pin policy* including wrong policy (they enshrine the change-type asymmetry) — so a security fix must update assertions, a process signal.

## Q17: Performance investigation method
**Anchor:** [05/04](../05-quality-engineering/04-performance-thinking.md); ACL subqueries.
**Mid adds:** reproduce with volume, measure (query count / EXPLAIN / queue latency), fix the biggest term, add the measurement as a regression test.
**Senior adds:** the ACL list query's nested `IN` + OR-over-inheritance is the likely hotspot at scale; index the ACL join, cache the role-short-circuit, materialize only if measured.

## Q18: Securing a file-processing pipeline
**Anchor:** [05/05 security checklist](../05-quality-engineering/05-security-checklist.md).
**Mid adds:** path-traversal neutralization ([document_models.py L233](../../../mayan/apps/documents/models/document_models.py#L230-L234)), server-side MIME detection, archive-bomb bounds ([expand recursion](../../../mayan/apps/documents/models/document_models.py#L205-L224)), sandbox the converter/OCR.
**Senior adds:** scanning as a pluggable pre-create hook ([project 4](../06-contribution-practice/03-senior-build-projects.md)); untrusted input reaches tesseract/soffice — isolate that worker tier.

**Self-grade:** *Strong* on this set means you can deliver Q6, Q8, Q9, and Q10 cold with anchors, and take each two follow-ups deep. Those four are the mid-level fullstack core.
