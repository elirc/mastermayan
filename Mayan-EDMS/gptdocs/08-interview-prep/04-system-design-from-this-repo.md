# System design: build a document management system

Prompt: design a service that uploads, versions, previews, OCRs, searches, protects, and audits business documents.

## 1. Requirements

Functional: upload/version/download, metadata and categories, OCR/search, workflow, roles and per-object permissions. Non-functional: durable bytes, consistent metadata, authorization on every access path, async processing, auditability, bounded recovery time. The repository advertises the product surface at [README.rst](../../README.rst#L9-L24).

Junior answers list screens. Mid-level answers separate synchronous acceptance from asynchronous processing and state security invariants. Senior answers negotiate volume, file size, compliance, retention, tenancy, RPO/RTO, and cost before selecting components.

## 2. API and contract

Use `POST /documents/{id}/files` with multipart data, return `202` plus a status resource, and expose idempotency keys. Mayan's actual view validates, stages, queues, and returns 202 at [document_file_api_views.py](../../mayan/apps/documents/api_views/document_file_api_views.py#L28-L60); its serializer exposes create-only versus read-only fields at [document_file_serializers.py](../../mayan/apps/documents/serializers/document_file_serializers.py#L71-L140).

Tradeoff: synchronous creation is simpler and gives immediate errors; async creation protects latency and worker capacity but needs progress and failure UX.

## 3. Data model

Core entities: DocumentType → Document → DocumentFile → pages/versions, plus User/Group/Role → ACL → object, and workflow template → instance/state. Binary bytes belong in object storage; relational metadata and state belong in the database. Mayan's file model relates a file to a document and records timestamp, checksum, size, MIME type, and storage field at [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L51-L129).

Invariant: a file row must not claim durable availability when its blob is missing. Because storage and SQL lack a shared transaction, use staged status plus reconciliation/compensation.

## 4. Processing architecture

Request → authorize → validate → stage blob → create job/outbox → worker → persist derived state → emit event. Mayan passes stable IDs to a worker at [documents/tasks.py](../../mayan/apps/documents/tasks.py#L48-L89). OCR fans out per page and joins completion via a chord at [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L17-L48).

At 10× traffic: apply file-size limits, backpressure, per-tenant quotas, queue partitioning, autoscaling by queue age, bounded retries with dead letters, and OCR concurrency limits. Measure p95 acceptance latency, queue age, processing duration, failure rate, and orphan count.

## 5. Security

Authentication establishes identity; authorization decides action on resource. Mayan separates view and object permissions at [rest_api/permissions.py](../../mayan/apps/rest_api/permissions.py#L9-L59) and scopes access through ACL querysets at [acls/managers.py](../../mayan/apps/acls/managers.py#L233-L294). Add malware scanning, content-type distrust, signed short-lived downloads, encryption, audit retention, and tenant-scoped keys where requirements demand them.

## 6. Reliability and rollout

Use an idempotency key for uploads and deterministic OCR work IDs. Never retry every exception. Keep staging blobs until durable completion; expire them with a reconciler. Roll out schema changes expand-first, deploy dual-read/write if needed, backfill observably, then contract. Roll back code without deleting newly written compatible data.

## Variations

1. Multi-tenancy: add tenant ownership to every resource and make tenant scope an invariant in query construction.
2. Real-time progress: publish job events to SSE/WebSocket; preserve polling fallback.
3. 10× uploads: direct-to-object-storage multipart upload plus finalize endpoint.
4. Legal hold: retention rules must override ordinary deletion and be auditable.
5. Regional residency: route storage and processing by tenant region; prevent cross-region task dispatch.

Practice rubric: **Junior** identifies components. **Mid** gives a concrete API/data model, one tradeoff, one failure mode, and a test. **Senior** states invariants, capacity assumptions, migration/rollback, observability, security, and when a simpler design wins.

Cross-read [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) before a mock interview.
