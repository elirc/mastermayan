# Testing strategy

## The layer model as practiced here

| Layer | Base class | What belongs | Example |
| --- | --- | --- | --- |
| Model/unit | `GenericDocumentTestCase` ([documents/tests/base.py L11–L12](../../../mayan/apps/documents/tests/base.py#L11-L12)) | invariants, state transitions, manager contracts | [test_document_models.py](../../../mayan/apps/documents/tests/test_document_models.py) |
| API view | `GenericDocumentAPIViewTestCase` ([base.py L15–L16](../../../mayan/apps/documents/tests/base.py#L15-L16)) | status codes, authz pairs, event assertions | [test_document_api.py L27–L66](../../../mayan/apps/documents/tests/test_document_api.py#L27-L66) |
| UI view | `GenericDocumentViewTestCase` | same discipline for template views | `test_document_views.py` |
| Migration | `MayanMigratorTestCase` ([testing/tests/base.py L83–L88](../../../mayan/apps/testing/tests/base.py#L83-L88)) | schema/data migration behavior | `test_migrations.py` per app |
| Cross-DB / CI | Makefile targets against dockerized Postgres/MySQL/Oracle ([Makefile L18–L22](../../../Makefile#L18-L22)) | engine-specific SQL behavior | |

Notably absent: browser/E2E (a Selenium mixin exists, [testing/tests/mixins.py L357](../../../mayan/apps/testing/tests/mixins.py), sparsely used) and load tests. The pyramid here is *fat in the middle* — view-level integration tests dominate. That's a legitimate strategy for a server-rendered monolith: the view layer is where the composition (ACL filter + queryset + serializer) can break.

## The house convention: the authz pair + event ledger

Every mutating endpoint gets at least: `test_X_no_permission` → 404, **zero events**, **zero state change**; `test_X_with_access` → 2xx, exact event count with exact actor/action_object/target/verb ([test_document_api.py L27–L66](../../../mayan/apps/documents/tests/test_document_api.py#L27-L66)). Three things ride along for free: authorization proof, side-effect accounting (nothing fired that shouldn't), and audit-contract pinning. When you add an endpoint here, writing anything less is an incomplete PR.

Trashed-object behavior gets a *third* member (`test_trashed_document_X_with_access`, [L114–L137](../../../mayan/apps/documents/tests/test_document_api.py#L114-L137)) because the valid-manager boundary is itself policy ([Flow 8](../01-codebase-cartography/05-key-flows.md)).

## Harness machinery worth stealing (rare in the wild)

- **Leak detectors as mixins**: open file descriptors ([testing/tests/mixins.py L250](../../../mayan/apps/testing/tests/mixins.py)), temp files (L427), and DB connection counts (L97) checked per test — the suite polices *resource hygiene*, not just assertions.
- **Random primary keys** (L273): monkeypatches models so tests never pass by accident of `pk=1` — kills a whole class of hardcoded-ID bugs.
- **Fixture-less object mamas**: `DocumentTestMixin` and friends create objects through real code paths (uploads run the actual save pipeline), so tests exercise the event/hook machinery instead of bypassing it with fixtures.
- **Isolation of time/randomness**: expiry tests (checkouts) construct absolute datetimes rather than sleeping.

## What not to test (house-consistent answers)

Django/DRF internals (trust the framework); storage engines (behind `DefinedStorage`, use the default test storage); Celery *transport* (call task functions synchronously — the repo's tests invoke behavior, not brokers); and rendering pixel output (converter tests assert page counts and formats, not images).

## Async testing rule

Anything behind `apply_async` is **eventually consistent** — tests must either call the task body directly or run eager mode; asserting right after the HTTP call flakes (see [Trace 5](../04-code-reading-gym/02-trace-tables.md)). Grep a search test (`test_document_search.py`) to see how index state is set up explicitly.

## Flake prevention checklist

- No `sleep`; use absolute datetimes and direct task invocation.
- No ordering assumptions on unordered querysets (Document default ordering is `label` — [document_models.py L126–L129](../../../mayan/apps/documents/models/document_models.py#L126-L129) — labels collide!).
- Registry pollution: tests that register events/permissions must use the provided test-model machinery ([TestModelTestCaseMixin, testing/tests/mixins.py L470](../../../mayan/apps/testing/tests/mixins.py)) which tears down synthetic models.

**Drill:** find the gap — this suite has strong authz pairs but (per grep in the [verification log](../09-reference/verification-log.md)) no direct view/API test for the version-modification endpoints. Write the test list you'd add for "append all file pages" *before* looking at [02-data-model's drill](../03-architecture-and-patterns/02-data-model-and-persistence.md) for why it matters. *Strong:* your list includes a model-level test that would catch the suspect `order_by` without any HTTP.

**Interview angle:** "How do you test authorization?" and "unit vs integration balance for a monolith?" — you now hold a concrete, defensible position with examples. Cards [08/03 Q16](../08-interview-prep/03-api-and-data-modeling-questions.md), behavioral story 6 ([08/06](../08-interview-prep/06-behavioral-star-stories.md)).
