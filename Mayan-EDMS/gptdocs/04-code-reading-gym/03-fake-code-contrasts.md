# Fake-code contrasts

All snippets are illustrative fake code, not from this repo.

1. `# Illustrative fake code: not from this repo` `DocumentFile.objects.get(pk=id)` → scope through `authorized_document.files.get(pk=id)`; compare [parent scoping](../../mayan/apps/documents/api_views/document_file_api_views.py#L89-L120).
2. `# Illustrative fake code: not from this repo` accept `document_id` from request body → derive parent from authorized URL resource.
3. `# Illustrative fake code: not from this repo` send uploaded bytes through broker → stage bytes and enqueue a stable ID; compare [upload](../../mayan/apps/documents/api_views/document_file_api_views.py#L44-L60).
4. `# Illustrative fake code: not from this repo` catch and retry every exception → retry transient classes, fail permanent validation, surface terminal state.
5. `# Illustrative fake code: not from this repo` emit email then save DB row → commit state/outbox, then deliver idempotently.
6. `# Illustrative fake code: not from this repo` loop pages and OCR serially → bounded fan-out/fan-in; compare [OCR chord](../../mayan/apps/ocr/tasks.py#L30-L43).
7. `# Illustrative fake code: not from this repo` load every file into memory for hashing → stream fixed blocks; compare [checksum](../../mayan/apps/documents/models/document_file_models.py#L186-L213).
8. `# Illustrative fake code: not from this repo` return 200 immediately after enqueue → return 202 with operation status.
9. `# Illustrative fake code: not from this repo` TypeScript `payload: any` → `unknown` + runtime parse + discriminated union.
10. `# Illustrative fake code: not from this repo` cache by `documentId` only → include identity/tenant/permission epoch or cache only public data.

Drill: for each, name the invariant protected and one situation where the “better” shape is unnecessary complexity.
