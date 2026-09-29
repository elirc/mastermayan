# Verification Log

Date: `2026-05-18 16:37:38 -07:00`

## Commands run

| Command | Result | Notes |
| --- | --- | --- |
| `rg --files` | Passed | Used for full inventory. |
| `git status --short` | Passed | Worktree clean enough for docs-only work. |
| `python --version` | Passed | `Python 3.13.3` |
| `pip --version` | Passed | `pip 25.0.1` |
| `python manage.py check --settings=mayan.settings.testing.development` | Failed | `ModuleNotFoundError: No module named 'django'` |

## Files inspected

- Root metadata: [`README.md`](../../README.md), [`CONTRIBUTING.md`](../../CONTRIBUTING.md), [`Makefile`](../../Makefile), [`tox.ini`](../../tox.ini), [`.gitlab-ci.yml`](../../.gitlab-ci.yml)
- Runtime/bootstrap: [`manage.py`](../../manage.py), [`mayan/settings/base.py`](../../mayan/settings/base.py), [`mayan/celery.py`](../../mayan/celery.py), [`docker/docker-compose.yml`](../../docker/docker-compose.yml)
- Core flows:
  [`mayan/apps/sources/source_backends/web_form_backends.py`](../../mayan/apps/sources/source_backends/web_form_backends.py),
  [`mayan/apps/sources/tasks.py`](../../mayan/apps/sources/tasks.py),
  [`mayan/apps/sources/models.py`](../../mayan/apps/sources/models.py),
  [`mayan/apps/documents/models/document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py),
  [`mayan/apps/documents/models/document_models.py`](../../mayan/apps/documents/models/document_models.py),
  [`mayan/apps/documents/models/document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py)
- Auth/permissions/API:
  [`mayan/apps/acls/models.py`](../../mayan/apps/acls/models.py),
  [`mayan/apps/rest_api/api_view_mixins.py`](../../mayan/apps/rest_api/api_view_mixins.py),
  [`mayan/apps/documents/api_views/document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py),
  [`mayan/apps/authentication/django_authentication_backends.py`](../../mayan/apps/authentication/django_authentication_backends.py),
  [`mayan/apps/authentication/views/authentication_views.py`](../../mayan/apps/authentication/views/authentication_views.py)
- Async flows:
  [`mayan/apps/file_metadata/methods.py`](../../mayan/apps/file_metadata/methods.py),
  [`mayan/apps/file_metadata/tasks.py`](../../mayan/apps/file_metadata/tasks.py),
  [`mayan/apps/document_parsing/methods.py`](../../mayan/apps/document_parsing/methods.py),
  [`mayan/apps/document_parsing/tasks.py`](../../mayan/apps/document_parsing/tasks.py),
  [`mayan/apps/ocr/tasks.py`](../../mayan/apps/ocr/tasks.py),
  [`mayan/apps/document_indexing/handlers.py`](../../mayan/apps/document_indexing/handlers.py)
- UI JavaScript:
  [`mayan/apps/appearance/static/appearance/js/mayan_app.js`](../../mayan/apps/appearance/static/appearance/js/mayan_app.js),
  [`mayan/apps/appearance/static/appearance/js/partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js),
  [`mayan/apps/tags/static/tags/js/tags_form.js`](../../mayan/apps/tags/static/tags/js/tags_form.js),
  [`mayan/apps/metadata/static/metadata/js/metadata_form.js`](../../mayan/apps/metadata/static/metadata/js/metadata_form.js)

## Uncertainties

- I did not verify live runtime behavior because dependencies are not installed in this environment.
- I did not execute Docker Compose profiles, Celery workers, or Selenium tests.
- Some areas such as `document_indexing`, `mailer`, and search backends were sampled rather than exhaustively traced.

## Coverage limits

- This suite emphasizes representative subsystems, not every app in `mayan/apps`.
- Legacy and generated translation files were intentionally not used as teaching anchors.
