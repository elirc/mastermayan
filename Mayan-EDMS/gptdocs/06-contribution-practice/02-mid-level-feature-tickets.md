# Mid-level feature tickets

Each requires a short design note, compatibility/risk section, observability, tests, and rollback.

1. **Upload operation status** (Hard): expose queued/running/completed/failed without claiming the DocumentFile exists early. Anchors: [202 view](../../mayan/apps/documents/api_views/document_file_api_views.py#L38-L60), [worker](../../mayan/apps/documents/tasks.py#L48-L124). Story: async UX and contract design.
2. **Staging orphan sweeper** (Medium): bounded cleanup with age/status safety. Story: reconciliation.
3. **Upload idempotency key** (Hard): prevent duplicate logical versions across retries. Story: distributed reliability.
4. **Cross-parent API matrix** (Medium): reusable negative tests for nested resources. Story: security automation.
5. **OCR partial-failure status** (Hard): per-page visibility and safe retry. Anchor: [chord](../../mayan/apps/ocr/tasks.py#L17-L48). Story: fan-out reliability.
6. **Queue-age metrics** (Medium): instrument upload/OCR/source tasks with bounded labels. Story: observability.
7. **Source action schema discovery** (Hard): formalize backend-specific argument schemas compatibly. Anchor: [dynamic serializer](../../mayan/apps/sources/serializers.py#L49-L63). Story: plugin contracts.
8. **ACL query benchmark** (Medium): representative fixture, query plans/count/latency, no premature rewrite. Story: performance method.
9. **Blob reconciliation command** (Hard): dry-run, report, repair modes with audit. Story: cross-store consistency.
10. **Safe download caching policy** (Hard): define identity-dependent cache rules. Story: performance/security tradeoff.

Maintainers reject designs without migration compatibility, permission model, failure state, targeted integration tests, and a rollback that works after new data has been written.
