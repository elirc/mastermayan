# Architecture critique

An honest review, written as if reporting to a CTO after two weeks in the code. Doubles as system-design interview prep — [08/04](../08-interview-prep/04-system-design-from-this-repo.md) turns this into a whiteboard session. Confirmed observations cite lines; hypotheses are labeled.

## Strongest design choices

1. **Authorization as a single queryset choke point** ([acls/managers.py L268–L294](../../../mayan/apps/acls/managers.py#L268-L294)) with inheritance and a 404 posture, *enforced by base classes* so the default path is the safe path ([rest_api/generics.py L20–L72](../../../mayan/apps/rest_api/generics.py#L20-L72)). Most CRUD apps never get this right; here it's structural.
2. **Files as immutable facts, versions as mutable views** ([document_version_models.py L315–L338](../../../mayan/apps/documents/models/document_version_models.py#L315-L338)) — audit-grade history, cheap undo, and page-level composition, from one modeling decision.
3. **Async-by-default with disciplined handoff** — IDs over the wire, staging tables, tiered workers, reapers ([03/04](04-side-effects-async-and-reliability.md)). The system degrades to "slower" rather than "down."
4. **The authz test-pair convention with event assertions** ([test_document_api.py L67–L112](../../../mayan/apps/documents/tests/test_document_api.py#L67-L112)) — policy is executable and regression-proof.
5. **Deployment-pluggable engines** (locks/search/storage) making one codebase serve laptop and cluster ([pattern 9](05-pattern-catalog.md)).

## Top risks and tradeoffs (prioritized)

| # | Risk | Evidence | Impact × likelihood | Mitigation path |
| --- | --- | --- | --- | --- |
| 1 | **ACL filter cost on large installs.** Every list request builds nested `IN` subqueries, recursively over inheritance; `check_access` repeats per permission | [acls/managers.py L31–L231, L233–L266](../../../mayan/apps/acls/managers.py#L31-L266) | high × likely at ~10⁶ docs (hypothesis — measure first) | query-count regression tests; cache role-wide short-circuit; consider materialized permission table (big RFC) |
| 2 | **Synchronous side effects inside saves/transitions** — event fan-out and notifications per save ([events/classes.py L387–L427](../../../mayan/apps/events/classes.py#L359-L429)); workflow actions inline ([workflow_instance_models.py L291–L309](../../../mayan/apps/document_states/models/workflow_instance_models.py#L282-L311)) | confirmed by reading | medium × certain (latency), high × occasional (failure coupling) | queue the fan-out; bound action time; document ordering guarantees |
| 3 | **Workflow transition race** — no lock/`select_for_update`; concurrent transitions both validate against the same current state | [workflow_instance_models.py L82–L104](../../../mayan/apps/document_states/models/workflow_instance_models.py#L82-L104) | medium × rare, but workflows guard *compliance* | expected-predecessor column + unique constraint; see design kata 2 |
| 4 | **Silent-failure surfaces**: fire-and-forget tasks with no user feedback (version modification, Flow from [02-data-model drill](02-data-model-and-persistence.md)); deindex DoesNotExist (*investigate*, [dynamic_search/tasks.py L27–L38](../../../mayan/apps/dynamic_search/tasks.py#L27-L38)); `except Exception` orderings ([ocr/tasks.py L114–L131](../../../mayan/apps/ocr/tasks.py#L114-L131)) | confirmed reads + 1 labeled investigate | medium × steady drip | dead-letter table; messaging-app notifications on task failure |
| 5 | **Authorization asymmetry on change-type** (user with edit on a doc can move it into any type, changing retention policy) — *test-enshrined*, so a decision; still risky combined with auto-delete periods | [document_serializers.py L65–L68](../../../mayan/apps/documents/serializers/document_serializers.py#L65-L68), [test L85–L112](../../../mayan/apps/documents/tests/test_document_api.py#L85-L112) | medium × depends on tenancy model | restrict target-type queryset by `document_create`; migration note: some workflows may rely on the loose behavior |
| 6 | **Global registries + import-order semantics** — startup fragility, test cross-talk risk | [settings/base.py L42–L48](../../../mayan/settings/base.py#L42-L48) | low × constant tax | contained by convention; document the ordering contract |
| 7 | **2022 dependency snapshot** (Django 3.2 LTS now EOL, `ugettext`, `url()`) | [requirements/base.txt](../../../requirements/base.txt) | context-dependent | upgrade path exists upstream; treat this copy as a study artifact |

## What I'd change owning this for 3 months

**Month 1 — visibility before change.** Query-count + latency regression harness on the five hottest list views (the test suite already counts connections — `ConnectionsCheckTestCaseMixin`, [testing/tests/mixins.py L97](../../../mayan/apps/testing/tests/mixins.py)); queue-depth and reap-count metrics; error-log growth dashboards. *Test strategy:* pin current behavior first, so every later change diffs against a baseline. *Rollback:* none needed — additive.

**Month 2 — de-risk the sharp edges.** Fix exception-ordering bugs and the `ignore_results` typo (one-liners with tests); add the change-type target-type restriction behind a setting default-off, flip after a release note (*migration path:* setting → default-on → remove setting); introduce a `FailedTask` dead-letter row + admin view. *Blast radius:* each change is one app; the setting gate makes the authz change reversible.

**Month 3 — one structural bet.** Either (a) async event fan-out (move notification creation to a queued task; contract: audit row still sync, notifications eventually consistent — needs an ordering note in docs), or (b) ACL materialization spike with benchmarks. Not both. Write the RFC first ([07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md) has the template).

## What I would *not* change

The event-decorator system (the smuggled actor is ugly but the alternative — threading `user` through 400 signatures — is uglier); the synchronous UI (right for the product); the manager taxonomy; per-app vertical slices. Knowing what to leave alone is the half of senior judgment nobody tests for — volunteer it.

**Drill:** pick risk #2 and write the one-page RFC: problem, evidence, two options, chosen option, migration, test plan, rollback. *Strong:* your migration section handles in-flight events during deploy and names the user-visible ordering change (notification may arrive after the page reload).
