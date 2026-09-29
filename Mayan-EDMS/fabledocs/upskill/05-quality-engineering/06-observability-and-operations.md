# Observability and operations

The organizing question for every flow: **"how would I know this broke, and how fast?"** This app's observability is uneven — strong per-entity error surfaces, thin aggregate metrics — which is itself the lesson: know what your system *can* tell you before an incident.

## What exists

| Signal | Where | Good for | Blind to |
| --- | --- | --- | --- |
| Python logging | `logger = logging.getLogger(__name__)` in every module; `logging` app configures it | tracing a single request/task if you have the logs | trends, rates, "how often" |
| Per-entity error logs | sources, document versions ([sources/tasks.py L47–L53](../../../mayan/apps/sources/tasks.py#L39-L53), OCR [ocr/tasks.py L44–L48](../../../mayan/apps/ocr/tasks.py#L44-L48)) | the *owner* seeing their failure in the UI | aggregate failure rate; cleared-on-success hides intermittency |
| Event/audit trail | `events` app — every mutation ([events/classes.py L359–L429](../../../mayan/apps/events/classes.py#L359-L429)) | forensics: who did what when | performance, infra health |
| Task manager UI | `task_manager` app lists queues/workers/task types ([task_manager/classes.py](../../../mayan/apps/task_manager/classes.py#L60-L208)) | seeing registered tasks, queue topology | live depth/latency unless the broker exposes it |
| Health/stats | `mayan_statistics` app; dashboards app | counts over time | real-time alerting |

## What's thin (name these gaps — they're senior signal)

- **No dead-letter visibility.** A task that exhausts retries logs and vanishes ([dynamic_search/tasks.py L60–L87](../../../mayan/apps/dynamic_search/tasks.py#L41-L89)); nothing counts it. Fix = senior project 4 ([06/03](../06-contribution-practice/03-senior-build-projects.md)).
- **No queue-depth/latency metrics** surfaced by the app — you rely on RabbitMQ/Redis tooling.
- **Silent fire-and-forget paths** (version modifications) have neither error log nor notification (Flow from [02-data-model drill](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
- **Storage/DB drift** (orphaned blobs, [Flow 4](../01-codebase-cartography/05-key-flows.md)) has no reconciliation report.

## "How would I know?" per key flow

| Flow | Broke how | Current detection | Better detection to propose |
| --- | --- | --- | --- |
| Upload (1) | worker down | user reports stub never resolves | `uploads` queue-age alert; stub-count gauge |
| Watch folder (2) | bad path | source error log (UI) | last-successful-scan timestamp + staleness alert |
| Authz (3) | over-broad grant | audit trail after the fact | periodic effective-permission report |
| OCR (5) | poison page | version error log, but finisher never clears | per-page failure counter; chord-timeout alert |
| Search (7) | index drift | none | periodic DB-count vs index-count diff |
| Trash (8) | reaper stuck | trash grows | reap-count metric = 0 alarm |

## Deploy and rollback (from the artifacts)

The deployable is a Docker image; CI runs the suite *inside* it before push ([.gitlab-ci.yml L43](../../../.gitlab-ci.yml#L36-L45)). Migrations run at container start (`initialsetup`/`performupgrade` entrypoints, inferred from [docker/](../../../docker/)). Rollback = redeploy prior image tag — **but** forward-only data migrations and in-flight queue messages complicate it ([02-data-model §schema change](../03-architecture-and-patterns/02-data-model-and-persistence.md)). The honest rollback rule: additive migrations roll back cleanly; anything with `RunPython` needs a reverse function or you can't `migrate` backward.

## The operator's first five minutes (incident playbook)

1. Which process class? (web 500s vs task backlog vs render errors)
2. Queue depths (broker UI) — backlog localizes to a worker tier.
3. Recent deploy? (image tag) — correlate with change.
4. Entity error logs for the affected feature (sources/versions).
5. Audit trail around the first bad timestamp.

**Drill:** for three flows above, write the exact metric (name, type, threshold) you'd add. E.g. `mayan_uploads_queue_oldest_message_age_seconds` (gauge, alert > 300). *Strong:* you tie each metric to a specific line of code that would emit it and note which are cheap (queue age from broker) vs invasive (index-drift diff needs a new periodic task).

**Interview angle:** "How do you know your system is healthy?" and "walk me through an incident" — the first-five-minutes playbook plus the gaps table is a complete, senior-flavored answer. Behavioral story: [08/06 story 5](../08-interview-prep/06-behavioral-star-stories.md).
