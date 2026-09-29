# API and data-modeling questions

## Q23: Why return 202 instead of 201 for upload?
Round: API/data. Repo anchor: [create response](../../mayan/apps/documents/api_views/document_file_api_views.py#L38-L42). Junior: async. Mid: 201 claims created resource; 202 needs status path. Senior: idempotency, expiry, terminal errors. Drill: design headers/body.

## Q24: Where should validation and authorization happen?
Round: API/data. Repo anchors: [serializer](../../mayan/apps/documents/serializers/document_file_serializers.py#L71-L140), [permission](../../mayan/apps/rest_api/permissions.py#L9-L59). Junior: controller. Mid: shape at boundary, invariant in domain, authorization before access. Senior: every path/bulk/job, TOCTOU. Drill: layer rules.

## Q25: How do nested URLs help and hurt?
Round: API/data. Repo anchor: [parent-scoped file view](../../mayan/apps/documents/api_views/document_file_api_views.py#L89-L120). Junior: show relationship. Mid: enforce parent-child consistency; longer URLs. Senior: canonical identity, authorization, moves, caching. Drill: wrong-parent test.

## Q26: Explain IDOR and the defense here.
Round: API/data. Repo anchors: [scoped queryset](../../mayan/apps/documents/api_views/document_file_api_views.py#L89-L120), [404 test](../../mayan/apps/documents/tests/test_document_file_api.py#L100-L106). Junior: guessing IDs. Mid: authorize object and relationship. Senior: list/search/export/cache/batch coverage. Drill: enumerate paths.

## Q27: How would you paginate an ACL-filtered list?
Round: API/data. Repo anchor: [authorized queryset](../../mayan/apps/acls/managers.py#L268-L294). Junior: limit/offset. Mid: filter before count/page; stable ordering. Senior: keyset pagination, permission changes, count cost. Drill: API contract.

## Q28: Why separate Document from DocumentFile?
Round: API/data. Repo anchor: [foreign key and metadata](../../mayan/apps/documents/models/document_file_models.py#L74-L120). Junior: versions. Mid: stable business identity versus byte artifacts. Senior: retention, workflow, dedupe, mutable metadata, migration. Drill: ERD.

## Q29: Can a DB transaction protect object storage changes?
Round: API/data. Repo anchor: [multi-store delete](../../mayan/apps/documents/models/document_file_models.py#L215-L236). Junior: no. Mid: compensation/reconciliation/status. Senior: saga/outbox, tombstones, legal hold, recovery SLO. Drill: inject failures.

## Q30: What is the purpose and limitation of a checksum?
Round: API/data. Repo anchor: [streaming SHA-256](../../mayan/apps/documents/models/document_file_models.py#L186-L213). Junior: detect changed bytes. Mid: dedupe/integrity, not identity. Senior: authenticity needs signature/MAC and trusted provenance. Drill: threat examples.

## Q31: How would you change this schema safely?
Round: API/data. Repo anchor: indexed file fields [model](../../mayan/apps/documents/models/document_file_models.py#L78-L120). Junior: migration. Mid: expand/backfill/contract and compatible deploy. Senior: lock/index strategy, old workers, rollback after writes. Drill: add status.

## Q32: How should dynamic source actions be modeled?
Round: API/data. Repo anchor: [action metadata/JSON](../../mayan/apps/sources/serializers.py#L27-L63). Junior: JSON arguments. Mid: per-action schema and allowlist. Senior: plugin versioning, discovery, backward compatibility, security. Drill: OpenAPI shape.
