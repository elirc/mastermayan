# Python runtime model

Python imports execute module top-level code once per process and cache modules. Django then populates an app registry; tasks often use `apps.get_model` to delay model resolution, as in [documents/tasks.py](../../mayan/apps/documents/tasks.py#L56-L72). Unlike Node promises, ordinary Python functions block the current worker; concurrency comes from multiple web/worker processes and Celery tasks. A Celery chord provides explicit parallel fan-out/fan-in at [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L30-L43).

Pitfalls: import cycles, shared mutable class registries, blocking I/O in web requests, catching `Exception`, and assuming retries undo side effects. Drill: mark every potentially blocking operation in the upload worker. Strong: separate DB, storage, and broker failure domains.

## Interview angle

Practice questions Q1–Q4 in [runtime deep dive](../08-interview-prep/01-js-ts-node-deep-dive.md): import lifecycle, blocking concurrency, exception boundaries, and worker payloads.
