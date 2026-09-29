# Side Effects, Async, and Reliability

## Side-effect map

| Side effect | Entry point | Worker/owner | Reliability note |
| --- | --- | --- | --- |
| upload processing | [`web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L57-L70) | [`sources/tasks.py`](../../mayan/apps/sources/tasks.py#L58-L103) | temp upload + retry on `OperationalError` |
| metadata extraction | [`file_metadata/methods.py`](../../mayan/apps/file_metadata/methods.py#L14-L28) | [`file_metadata/tasks.py`](../../mayan/apps/file_metadata/tasks.py#L17-L48) | lock-based de-duplication |
| parsing | [`document_parsing/methods.py`](../../mayan/apps/document_parsing/methods.py#L37-L49) | [`document_parsing/tasks.py`](../../mayan/apps/document_parsing/tasks.py#L11-L34) | simple handoff |
| OCR | task-triggered | [`ocr/tasks.py`](../../mayan/apps/ocr/tasks.py#L17-L131) | chord, per-page retry, finish event |
| indexing | signals/handlers | [`document_indexing/handlers.py`](../../mayan/apps/document_indexing/handlers.py#L39-L83) | task storm risk |

## Concepts to learn here

- Idempotency:
  lock-based file metadata processing is a practical example.
- Retries:
  upload and OCR selectively retry infrastructure failures.
- Backpressure:
  OCR page fan-out can multiply load by page count.
- Failure visibility:
  OCR and parsing use error logs instead of silent failure.

## Risky places

- Doing expensive work inside `DocumentFile.save()` is convenient but increases transaction and request-path weight.
- Signal-triggered indexing can be hard to reason about during bulk operations.
