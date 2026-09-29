# Runtime and tooling map

| Surface | Evidence | Command |
| --- | --- | --- |
| Web | Django + DRF dependencies: [setup.py](../../setup.py#L63-L85) | `make runserver` — inferred |
| Worker | Celery app autodiscovers installed apps: [celery.py](../../mayan/celery.py#L7-L11) | deployment-specific — not verified |
| Tests | Make supports targeted module tests | `make test MODULE=...` — inferred |
| Docs | Make has HTML/live/spellcheck targets | `make docs-html` — inferred |
| Data | Django DB config; production commonly PostgreSQL | `make manage COMMAND="migrate"` — inferred |

Runtime boundaries are WSGI/ASGI web processes, Celery workers/beat, relational DB, cache/broker, search service, and file storage. Environment settings select databases, queues, storage, and integrations; never paste secret values into docs or issues. The base serializer uses JSON task payloads at [base.py](../../mayan/settings/base.py#L287-L300).

Sharp edge: `tox.ini` appears materially older than `setup.py`; confirm supported toolchain before treating it as authoritative.
