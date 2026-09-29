# Senior build projects

## Project 1: durable async operation model (1–3 weeks)
Value: users and operators see terminal upload/OCR states. Decide operation schema, idempotency, retention, permission, metrics, migration, API compatibility, and rollback. Anchors: [upload handoff](../../mayan/apps/documents/api_views/document_file_api_views.py#L44-L60), [OCR fan-out](../../mayan/apps/ocr/tasks.py#L17-L48). Interview story: leading a cross-layer reliability design.

## Project 2: transactional outbox for critical jobs (2–4 weeks)
Value: closes DB-to-broker handoff gap. Include publisher ownership, dedupe, ordering, poison records, observability, backfill, dual-publish migration, and rollback. Story: consistency versus complexity.

## Project 3: authorization conformance suite (1–2 weeks)
Value: systematic list/detail/bulk/parent scoping. Generate a permission matrix without hiding bespoke semantics. Anchor: [permission adapter](../../mayan/apps/rest_api/permissions.py#L9-L59). Story: reducing security blast radius.

## Project 4: blob lifecycle reconciler (2–3 weeks)
Value: converges DB/storage/cache after partial failure. Dry-run first; rate limit; legal hold; audit; deletion safety; restore. Anchor: [delete order](../../mayan/apps/documents/models/document_file_models.py#L215-L236). Story: operational safety.

## Project 5: OCR capacity and fairness (1–3 weeks)
Value: prevent large documents/tenants monopolizing workers. Model page cost, queue partition, limits, priorities, backpressure, SLOs, and partial retry. Story: scaling with fairness.

## Project 6: supported-toolchain modernization RFC (2 days design, weeks rollout)
Value: align packaging, CI, Tox, and docs. Inventory generated sources, dependencies, native images, migrations, deprecation policy, and rollback. Story: migration leadership.

All projects need product value, decision log, threat model, load/failure tests, rollout gates, dashboards, rollback, and explicit open questions. Stretch goal: teach the design with a small maintainer-facing runbook.
