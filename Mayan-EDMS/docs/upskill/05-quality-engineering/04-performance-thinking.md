# Performance Thinking

Measure first.

## Likely hotspots

- upload path derived work in [`document_file_models.py`](../../../mayan/apps/documents/models/document_file_models.py#L460-L484)
- OCR page fan-out in [`ocr/tasks.py`](../../../mayan/apps/ocr/tasks.py#L30-L43)
- index update task bursts in [`document_indexing/handlers.py`](../../../mayan/apps/document_indexing/handlers.py#L71-L83)
- client-side rapid AJAX navigation in [`partial_navigation.js`](../../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L96-L115)

## What to look for

- serial async work where parallelism is safe
- unbounded archive expansion during upload
- expensive save-time computation
- permission filters causing heavy generic-content-type joins

## Drill

- Pick one hotspot and propose a measurement plan before proposing a fix.
