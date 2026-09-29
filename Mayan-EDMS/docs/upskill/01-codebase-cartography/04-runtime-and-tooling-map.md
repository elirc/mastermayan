# Runtime and Tooling Map

## Toolchain

| Concern | Evidence | What it means |
| --- | --- | --- |
| Python entrypoint | [`manage.py`](../../manage.py#L7-L18) | Standard Django CLI entry. |
| Main dependency set | [`requirements/base.txt`](../../requirements/base.txt#L1-L47) | Django, DRF, Celery, OCR/search helpers, Whitenoise. |
| Test-only deps | [`requirements/testing-base.txt`](../../requirements/testing-base.txt#L1-L6) | migration tests, Selenium, coverage. |
| Command hub | [`Makefile`](../../Makefile#L10-L551) | tests, staging, docs, DB containers, safety, build. |
| CI | [`.gitlab-ci.yml`](../../.gitlab-ci.yml#L249-L346) | SQLite + Postgres tests, upgrade tests, docs build. |

## Runtime boundaries

| Boundary | Files | Notes |
| --- | --- | --- |
| Browser HTML app | app `views.py`, templates, static JS | Partial navigation replaces full-page assumptions in many places. |
| API | [`rest_api/urls.py`](../../mayan/apps/rest_api/urls.py#L10-L45) | Versioned endpoints plus auth token endpoint. |
| Worker | [`mayan/celery.py`](../../mayan/celery.py#L1-L11) | Celery autodiscovers tasks from installed apps. |
| Database | Django ORM + migrations | Default local fallback is SQLite if DB env not configured. |
| Cache/lock | Redis in Compose | Also used as lock backend in Compose profile. |

## Environment and secrets

- Secret key loading:
  [`mayan/settings/base.py`](../../mayan/settings/base.py#L21-L33) loads from `MAYAN_SECRET_KEY` or a file in `MEDIA_ROOT`.
- Database fallback:
  [`mayan/settings/base.py`](../../mayan/settings/base.py#L284-L304) falls back to SQLite if DB env vars are absent.
- Compose secrets by environment variables:
  [`docker/docker-compose.yml`](../../docker/docker-compose.yml#L5-L12) configures broker, Redis, database, and lock backend.

## Local/staging shapes

- App-only:
  Compose `app` profile runs all-in-one [`docker/docker-compose.yml`](../../docker/docker-compose.yml#L29-L37).
- Split runtime:
  Compose has `frontend`, worker classes, `celery_beat`, and infra services [`docker/docker-compose.yml`](../../docker/docker-compose.yml#L121-L252).
- Makefile staging:
  `staging-start`, `staging-frontend`, and `staging-worker` simulate production-like local separation [`Makefile`](../../Makefile#L418-L431).

## Verification Notes

- Runtime map is based on settings, Makefile, Compose, and CI config.
- No containers were launched in this pass.
