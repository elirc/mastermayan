# Fast Track: One Weekend with Mayan EDMS

This path is for a fast, real win. It does not try to make you a maintainer in two days; it gets you from "fresh clone" to "I can follow a flow and propose a safe change."

## Install, run, test

### Local commands

- Verified:
  `python --version` -> Python `3.13.3`
- Verified:
  `pip --version` -> `pip 25.0.1`
- Attempted:
  `python manage.py check --settings=mayan.settings.testing.development`
  Result: failed because Django is not installed yet, which matches the dependency-driven setup in [`manage.py`](../../manage.py#L7-L18) and [`requirements/base.txt`](../../requirements/base.txt#L1-L47).

### Recommended setup path

- Inferred from [`README.md`](../../README.md#L44-L51) and [`docker/docker-compose.yml`](../../docker/docker-compose.yml#L3-L289):
  Use Docker first if you want the easiest "whole system" path.
- Inferred from [`Makefile`](../../Makefile#L472-L489):
  For local Python development, install the dev, test, docs, and build requirements.

```bash
# Inferred from Makefile targets.
pip install --requirement requirements.txt \
  --requirement requirements/development.txt \
  --requirement requirements/testing-base.txt \
  --requirement requirements/documentation.txt \
  --requirement requirements/build.txt
```

```bash
# Inferred from Makefile.
python manage.py runserver --settings=mayan.settings.development
python manage.py test --mayan-apps --settings=mayan.settings.testing.development --skip-migrations
```

## Two flows to trace

### Flow 1: Web upload to document creation

Open these files first:

- [`mayan/apps/sources/source_backends/web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L28-L72)
- [`mayan/apps/sources/tasks.py`](../../mayan/apps/sources/tasks.py#L58-L103)
- [`mayan/apps/sources/models.py`](../../mayan/apps/sources/models.py#L87-L161)
- [`mayan/apps/documents/models/document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L138-L176)
- [`mayan/apps/documents/models/document_models.py`](../../mayan/apps/documents/models/document_models.py#L189-L251)
- [`mayan/apps/documents/models/document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py#L428-L505)

What to notice:

- ACL filtering happens before the upload task is queued.
- Upload is staged through `SharedUploadedFile`, not passed inline forever.
- Document creation and document-file creation are separate failure boundaries.
- Post-upload signals and callbacks trigger follow-on work.

### Flow 2: Document file to async parsing and metadata

Open these files first:

- [`mayan/apps/file_metadata/methods.py`](../../mayan/apps/file_metadata/methods.py#L7-L28)
- [`mayan/apps/file_metadata/tasks.py`](../../mayan/apps/file_metadata/tasks.py#L17-L48)
- [`mayan/apps/document_parsing/methods.py`](../../mayan/apps/document_parsing/methods.py#L16-L49)
- [`mayan/apps/document_parsing/tasks.py`](../../mayan/apps/document_parsing/tasks.py#L11-L34)
- [`mayan/apps/ocr/tasks.py`](../../mayan/apps/ocr/tasks.py#L17-L131)

What to notice:

- Submission methods are intentionally thin.
- Worker tasks recover user context by `user_id`.
- File metadata uses a per-file lock to avoid concurrent duplicate processing.
- OCR is split into page tasks plus a finishing callback.

## First 10 files to open

1. [`README.md`](../../README.md)
2. [`CONTRIBUTING.md`](../../CONTRIBUTING.md)
3. [`Makefile`](../../Makefile)
4. [`mayan/settings/base.py`](../../mayan/settings/base.py#L34-L123)
5. [`mayan/celery.py`](../../mayan/celery.py)
6. [`mayan/apps/documents/urls.py`](../../mayan/apps/documents/urls.py#L95-L688)
7. [`mayan/apps/rest_api/urls.py`](../../mayan/apps/rest_api/urls.py#L10-L45)
8. [`mayan/apps/sources/source_backends/web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py#L28-L72)
9. [`mayan/apps/documents/models/document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L138-L176)
10. [`mayan/apps/documents/models/document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py#L428-L505)

## One small safe change to attempt

Add or improve a docs-only note that clarifies one permission boundary or one async step in the upload path. A low-risk product-code alternative, if maintainers want it, would be improving an error message or adding a missing test around a permission-denied upload case in [`mayan/apps/sources/tests/test_web_form_source_api.py`](../../mayan/apps/sources/tests/test_web_form_source_api.py).

## One check to run

If dependencies are installed:

```bash
# Inferred from Makefile and test names.
python manage.py test mayan.apps.sources.tests.test_web_form_source_api \
  --settings=mayan.settings.testing.development --skip-migrations
```

## Teach-back exercise

Explain, without looking, why Mayan EDMS queues uploads instead of always creating documents synchronously in the request path. Your answer should mention authorization, large files, retry behavior, and why `SharedUploadedFile` exists.

## What this fast path does not cover

- Deep ACL inheritance mechanics.
- Search and indexing internals.
- OCR backend configuration and failure recovery in production.
- Role design, RFC writing, and architecture critique.

## Verification Notes

- Commands actually run in this workspace: `python --version`, `pip --version`, `python manage.py check --settings=mayan.settings.testing.development`.
- The Django command failed because dependencies are not installed here.
- All other run/test commands above are inferred from [`Makefile`](../../Makefile#L10-L551) and settings files.
