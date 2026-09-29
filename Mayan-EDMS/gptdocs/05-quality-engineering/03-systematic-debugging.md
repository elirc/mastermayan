# Systematic debugging

Workflow: reproduce → narrow layer → state one falsifiable hypothesis → use the cheapest probe → fix root cause → add regression coverage.

## Scenario: upload stays pending
Reproduce with one small file. Is failure before or after 202? Check response, staging row, broker enqueue, queue age, worker log, then task cleanup. Likely causes: worker unavailable, stale IDs, storage error, retry loop. Regression: operation reaches completed/failed. Senior lesson: async success needs terminal visibility.

## Scenario: authorized download returns 404
Compare parent ID, child membership, method permission, role/global permission, ACL inheritance, and trashed state. Use queryset inspection, not random permission changes. Regression: permission matrix plus wrong-parent case.

## Scenario: OCR never finishes
Inspect chord creation, page count, cache image, per-page retries, result backend, callback. The cache-miss retry is explicit at [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L75-L89). Regression: one transient page failure then completion.

## Scenario: source processes twice
Inspect lock name/TTL, task duration, worker clock, backend behavior, and duplicate scheduling at [sources/tasks.py](../../mayan/apps/sources/tasks.py#L18-L55). Regression: concurrent invocation produces one backend call.

## Scenario: deleted file blob remains
Trace pages/blob/cache/row order at [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L215-L236). Inject failure after each side effect. Regression: reconciler or retry converges.

Interview narration: say what you know, the boundary you are splitting, your next cheapest discriminating test, and what evidence would falsify the hypothesis.
