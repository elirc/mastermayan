# Verification log

Updated 2026-07-11 (America/Los_Angeles). This log was started before the curriculum was drafted.

| Inspection | Evidence/result | Confidence |
| --- | --- | --- |
| Root inventory | Django project with `mayan/apps`, Docker, docs, Makefile, setup metadata, GitLab CI | High |
| Product identity | Document management, OCR, preview, labels, signatures, workflows, RBAC, REST API: [README.rst](../../README.rst#L9-L24) | High |
| Runtime dependencies | Django 3.2.14, DRF 3.13.1, Celery 5.2.3, Elasticsearch 7.17.1: [setup.py](../../setup.py#L62-L110) | High |
| API upload | Validates input, stages the file, queues work, returns 202: [document_file_api_views.py](../../mayan/apps/documents/api_views/document_file_api_views.py#L28-L65) | High |
| Authorization | View/object permissions are separate; object checks delegate to ACLs: [permissions.py](../../mayan/apps/rest_api/permissions.py#L9-L59) | High |
| ACL behavior | Anonymous gets an empty queryset; global permission can allow the full queryset: [managers.py](../../mayan/apps/acls/managers.py#L233-L294) | High |
| Upload worker | Loads IDs, creates a document file, retries DB operational errors, deletes staging data: [tasks.py](../../mayan/apps/documents/tasks.py#L48-L124) | High |
| OCR fan-out | A Celery chord runs per-page OCR then a completion task: [tasks.py](../../mayan/apps/ocr/tasks.py#L17-L48) | High |
| Tests | Permission denial is deliberately represented as 404 and access success asserts data/events: [test_document_file_api.py](../../mayan/apps/documents/tests/test_document_file_api.py#L27-L74) | High |
| Commands | Make targets were inspected, not executed. `make test`, `make runserver`, and setup commands are inferred. | Medium |

## Commands run

- `Get-ChildItem` and targeted `rg -n` / `rg --files`: passed for inspected areas.
- `git status --short`: ran, but the detected Git root is `C:/Users/Owner`, not this project. Its output is therefore not a reliable project worktree status.
- Tests/build/server: not run. This Windows host has not been confirmed to have the historical native libraries, services, or supported Python environment.

## Uncertainties and limits

- `setup.py` pins Django 3.2 while `tox.ini` still describes Python 2.7/3.6 and Django 1.11. Treat Tox as legacy until maintainers confirm it.
- No claim here says a suspicious design is a confirmed bug. Risks are hypotheses with proposed tests.
- UI templates, every installed app, migrations, deployment images, search backends, and external integrations were sampled rather than exhaustively audited.
- The learner target is full-stack JS, but this repository is Python/Django. The curriculum explicitly translates concepts across stacks.
