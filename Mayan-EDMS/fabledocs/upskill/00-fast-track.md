# Fast track — one weekend with Mayan EDMS

Goal: by Sunday night you can run the app (or at least its test suite in Docker), trace two end-to-end flows aloud, and have made one safe change with a passing test.

## 1. Get it running (Saturday morning)

All commands are `inferred` from the repo's own scripts unless marked otherwise — the authoring machine (Python 3.13, Windows) cannot run this Django 3.2 / Celery 5.2 codebase natively. Two viable routes:

**Route A — Docker (recommended).** The compose file wires Postgres, Redis, RabbitMQ, and the app image, including the Redis-based distributed lock backend ([docker/docker-compose.yml](../../docker/docker-compose.yml#L3-L16)):

```
cd docker
cp .env.example .env   # inferred; check the docker/ README conventions
docker compose up -d
```

**Route B — local venv (Linux/WSL, Python 3.9-ish).** `inferred` from [setup.py](../../setup.py) and [Makefile](../../Makefile):

```
python3.9 -m venv venv && . venv/bin/activate
pip install -r requirements.txt
./manage.py initialsetup      # migrations + initial data; inferred from docs
./manage.py runserver
```

**Tests** run through the Makefile, which wraps `manage.py test` with a testing settings module and skips migrations by default ([Makefile](../../Makefile#L1-L60)):

```
make test MODULE=mayan.apps.documents     # one app
make test-all                             # everything (slow)
```

## 2. The first 10 files to open (Saturday afternoon)

Open these in order; each takes 5–15 minutes. Full reading order with junior/mid/senior paths: [01-codebase-cartography/02-file-reading-order.md](01-codebase-cartography/02-file-reading-order.md).

1. [mayan/__init__.py](../../mayan/__init__.py) — version, Django version (the repo-root `__init__.py` is empty; `__init__.py.tmpl` is the release template). You are reading a 2022-era snapshot; expect `ugettext_lazy` and `url()` regex routing.
2. [mayan/settings/base.py](../../mayan/settings/base.py#L1-L60) — settings are *generated* through a `SettingNamespaceSingleton`, not hand-written constants. Environment variables prefixed `MAYAN_` override them.
3. [mayan/apps/common/apps.py](../../mayan/apps/common/apps.py#L27-L120) — `MayanAppConfig.configure_urls()`: every app self-registers its URLs at startup into an initially **empty** [mayan/urls/base.py](../../mayan/urls/base.py). This one file explains how 57 apps compose into one site.
4. [mayan/apps/documents/models/document_models.py](../../mayan/apps/documents/models/document_models.py#L41-L108) — the `Document` model: UUID, type, `in_trash`, `is_stub`, and three managers (`objects`, `trash`, `valid`).
5. [mayan/apps/documents/models/document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L428-L504) — `DocumentFile.save()`: the busiest 70 lines in the repo. Checksum, MIME detection, page count, all inside a transaction.
6. [mayan/apps/acls/managers.py](../../mayan/apps/acls/managers.py#L268-L294) — `restrict_queryset()`: authorization as a queryset filter, the single most important security function here.
7. [mayan/apps/rest_api/generics.py](../../mayan/apps/rest_api/generics.py#L20-L72) — API base classes that bolt that ACL filter onto DRF views by default.
8. [mayan/apps/documents/tasks.py](../../mayan/apps/documents/tasks.py#L52-L124) — `task_document_file_upload`: the async half of every upload.
9. [mayan/apps/documents/queues.py](../../mayan/apps/documents/queues.py) — queue/task registration; cross-reference [task_manager/workers.py](../../mayan/apps/task_manager/workers.py#L12-L36) for the A–D worker tiers.
10. [mayan/apps/events/decorators.py](../../mayan/apps/events/decorators.py#L8-L33) — `@method_event`: how nearly every model save/delete emits an audit event.

## 3. Trace two flows aloud (Sunday morning)

Use the trace tables in [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md). Minimum: **Flow 2** (API upload → Celery → `DocumentFile.save`) and **Flow 5** (ACL check on a document detail request). Narrate each as if an interviewer asked "walk me through what happens when a user uploads a file" — 3 minutes, no notes. That teach-back *is* the exercise; record yourself and listen for hand-waving.

## 4. One small safe change + one test (Sunday afternoon)

Do Ticket 1 or Ticket 2 from [06-contribution-practice/01-good-first-tickets.md](06-contribution-practice/01-good-first-tickets.md) (both are one-line fixes with an obvious test). Run the focused suite (`inferred`):

```
make test MODULE=mayan.apps.documents.tests.test_document_models
```

If you can't get a runtime at all this weekend, do the change anyway and write the test by imitating an adjacent one — e.g. the checkout tests exercise model behavior without any HTTP ([checkouts/models.py](../../mayan/apps/checkouts/models.py#L100-L116) and its `tests/` sibling directory) — then verify on Monday in Docker.

## 5. What the fast path skips

Workflows (`document_states`), search backends, the converter/transformation pipeline, file caching + distributed locks, and all of quality/security. Those are modules 03 and 05. Skipping them this weekend is fine; claiming you know the repo without them is not.
