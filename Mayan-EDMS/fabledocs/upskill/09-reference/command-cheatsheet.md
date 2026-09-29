# Command cheat sheet

Status: `verified` = actually run against this working copy; `inferred` = read from the Makefile/CI/docker files but not executed (the norm — this Django 3.2 codebase does not run on the authoring machine's Python 3.13). See [verification-log](verification-log.md).

## Inspecting the repo (verified)

| Command | Purpose |
| --- | --- |
| `find mayan -name "*.py" \| wc -l` | file count (2,181) — `verified` |
| `ls mayan/apps` | enumerate the 57 apps — `verified` |
| `grep -rn "restrict_queryset" mayan/apps` | find every authz enforcement point — `verified` (works; scope to a subdir, full-tree ripgrep can time out) |

## Setup / run (inferred)

| Command | Purpose |
| --- | --- |
| `docker compose up -d` (in `docker/`) | full stack: app + Postgres + Redis + RabbitMQ ([docker-compose.yml](../../../docker/docker-compose.yml)) |
| `pip install -r requirements.txt` | install pinned deps ([requirements.txt](../../../requirements.txt)) |
| `./manage.py initialsetup` | migrations + initial data (first run) |
| `./manage.py runserver --settings=mayan.settings.development` | dev server (default settings module is `production`, [celery.py L7](../../../mayan/celery.py#L7)) |
| `./manage.py shell` | ORM/queryset probing (the debugging workhorse) |
| `celery -A mayan worker` | run a worker (see tiers in [workers.py](../../../mayan/apps/task_manager/workers.py#L12-L36)) |

## Testing (inferred from [Makefile](../../../Makefile#L1-L60))

| Command | Purpose |
| --- | --- |
| `make test MODULE=mayan.apps.documents` | one app |
| `make test MODULE=mayan.apps.documents.tests.test_document_models` | one module |
| `make test MODULE=...ClassName.test_method` | one test |
| `make test-all` | everything (slow) |
| `make test SKIPMIGRATIONS=` | run *with* migrations (default injects `--skip-migrations`) |
| `make test-debug MODULE=...` | debug mode |
| `make test-with-postgresql MODULE=...` | against dockerized Postgres ([Makefile L18–L22](../../../Makefile#L18-L22)) |
| `make test-with-mysql` / `test-with-oracle` | other engines |

Under the hood the test command is `./manage.py test $MODULE --settings=mayan.settings.testing.development --skip-migrations` ([Makefile L16](../../../Makefile#L1-L60)).

## Migrations (inferred)

| Command | Purpose |
| --- | --- |
| `./manage.py makemigrations <app>` | generate a migration after a model change |
| `./manage.py migrate` | apply |
| `./manage.py migrate <app> <number>` | migrate to a specific point (rollback) |
| `./manage.py sqlmigrate <app> <number>` | preview the SQL |

## Maintenance / ops (inferred)

| Command | Purpose |
| --- | --- |
| `./manage.py performupgrade` | run upgrade steps after a version bump |
| `./manage.py purgelocks` | clear stuck distributed locks (lock_manager) |
| `./manage.py document_ocr_submit` | (re)submit documents for OCR (management commands in `ocr/management/`) |
| `./manage.py search_index_rebuild` | rebuild the search index (dynamic_search management) — verify exact name in the app |

Management command names above are `inferred` from app conventions; confirm with `./manage.py help` before relying on them.

## Quick probes for debugging (inferred, run in `./manage.py shell`)

```python
from mayan.apps.documents.models import Document
Document.objects.filter(pk=X).values('is_stub', 'in_trash')   # not-yet vs trashed
from mayan.apps.acls.models import AccessControlList
AccessControlList.objects.restrict_queryset(               # reproduce authz
    permission=..., queryset=Document.valid.all(), user=...
).filter(pk=X).exists()
print(Document.valid.all().query)                          # inspect SQL
```

**Note:** every command touching a running system is `inferred` here. Before quoting one as fact in an interview, say "the Makefile defines…" rather than "I ran…" unless you actually ran it.
