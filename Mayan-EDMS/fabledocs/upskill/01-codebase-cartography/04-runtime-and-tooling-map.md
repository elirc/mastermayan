# Runtime and tooling map

## Processes at runtime

A production deployment runs **one web process + four Celery worker classes + a beat scheduler**, all from the same codebase:

| Process | What runs there | Why separated |
| --- | --- | --- |
| gunicorn (Django) | UI + REST views. Anything slow is *dispatched*, not done. | Latency: a request must never wait on OCR |
| `worker_a` (nice 0) | Interactive-adjacent tasks (e.g. page image generation) | User is actively waiting |
| `worker_b` (nice 2) | Medium tasks — document maintenance, version ops | |
| `worker_c` (nice 10) | Uploads, periodic scans | Bulky but tolerable delay |
| `worker_d` (nice 15) | Heaviest, latency-insensitive (OCR-class work) | Must not starve everything else |
| celery beat | Periodic schedules declared in each app's `queues.py` | |

Worker tiers: [task_manager/workers.py](../../../mayan/apps/task_manager/workers.py#L12-L36). Queue→worker binding happens where queues are declared, e.g. `queue_uploads` on worker_c and `queue_documents` on worker_b in [documents/queues.py](../../../mayan/apps/documents/queues.py#L14-L22). Periodic tasks are declared with `schedule=timedelta(...)` right there too ([documents/queues.py](../../../mayan/apps/documents/queues.py#L50-L69)) — there is no crontab file to find.

**Transferable concept:** this is *workload isolation by latency class* — the same idea as separate Node worker pools or K8s node pools. Interview vocabulary: **backpressure** (RabbitMQ queues absorb bursts), **blast radius** (a stuck OCR job can only clog worker_d's queues).

## Runtime boundaries

- **Browser**: server-rendered Django templates (Bootstrap, jQuery — see [appearance/dependencies.py](../../../mayan/apps/appearance/dependencies.py)). No SPA, no client state store. Partial updates via AJAX template endpoints.
- **Web process**: request/response only. Uploads are parked as `SharedUploadedFile` and continue in workers ([sources mixins](../../../mayan/apps/sources/source_backends/mixins.py#L61-L93)).
- **Workers**: receive primitive kwargs (IDs), re-fetch models via `apps.get_model(...)` ([documents/tasks.py](../../../mayan/apps/documents/tasks.py#L26-L36)) — never pickled model instances. Why: messages outlive code versions and web-process memory.
- **Shared state**: Postgres (source of truth), object storage (bytes), Redis (locks/result backend), RabbitMQ (broker) — wired in [docker/docker-compose.yml](../../../docker/docker-compose.yml#L3-L16).

## Settings

Settings are built at import time by `SettingNamespaceSingleton` from env vars ([settings/base.py](../../../mayan/settings/base.py#L18-L28)); `MAYAN_SECRET_KEY` env → file fallback under the media root ([settings/base.py](../../../mayan/settings/base.py#L30-L38)). Per-app knobs live in each app's `settings.py` (e.g. hash block size in `documents/settings.py`, consumed at [document_file_models.py](../../../mayan/apps/documents/models/document_file_models.py#L186-L197)). High-level env vars you'll actually touch: `MAYAN_DATABASES`, `MAYAN_CELERY_BROKER_URL`, `MAYAN_CELERY_RESULT_BACKEND`, `MAYAN_LOCK_MANAGER_BACKEND(_ARGUMENTS)` — all visible with sane defaults in the compose file. No secrets are committed; don't add any.

## Toolchain

| Task | Command | Status |
| --- | --- | --- |
| Run one app's tests | `make test MODULE=mayan.apps.documents` | inferred from [Makefile](../../../Makefile#L1-L60) |
| All tests | `make test-all` | inferred |
| Tests w/ migrations | `make test SKIPMIGRATIONS=` | inferred (default injects `--skip-migrations`) |
| Alt DB engines | `make test-with-mysql` / `-postgresql` / `-oracle` (dockerized DBs) | inferred from Makefile targets + container names |
| Dev server | `./manage.py runserver` (settings default to `mayan.settings.production`; use `--settings=mayan.settings.development`) | inferred |
| Docker stack | `docker compose up` in `docker/` | inferred |
| CI | GitLab CI: test → build python → build docker → docs → push ([.gitlab-ci.yml](../../../.gitlab-ci.yml#L1-L9)) | read, not run |
| Packaging | `setup.py` generated from `setup.py.tmpl` | inferred |

There is **no ESLint/Prettier equivalent enforced in-repo** (flake8-style conventions are followed by hand: single-quote strings, keyword-argument-always calls, alphabetized imports). Match the house style; a maintainer will bounce a PR that doesn't. More in [07-career/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).

**Drill:** from [documents/queues.py](../../../mayan/apps/documents/queues.py) alone, answer: which worker handles emptying the trash? Which handles page-count updates? What's the schedule for stub cleanup? *Basic:* read the three answers off the file. *Solid:* explain why trash-emptying fans out one task per document ([documents/tasks.py](../../../mayan/apps/documents/tasks.py#L299-L311)) instead of one big loop — failure isolation and parallelism. *Strong:* say what happens to in-flight tasks when you deploy new code (hidden contract: old messages, new consumers).

**Interview angle:** "How does your system handle long-running work?" — answer with this topology, then the SharedUploadedFile handoff, then the deploy-boundary caveat. Cards in [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) and [04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).
