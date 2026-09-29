# Observability and Operations

## What exists

- app logging in settings/middleware and task modules
- OCR/parsing error-log models registered by app config
- CI upgrade tests and DB matrix

## How would I know this broke?

- Upload: missing documents after `202`, orphan temp uploads, worker logs.
- Parsing: document preview exists but searchable/extracted text missing.
- OCR: page tasks retry repeatedly or version error logs accumulate.
- Auth: public login flow redirects incorrectly or MFA session handoff loops.

## Local vs prod difference

- Local fallback can use SQLite from settings, while Compose points to Postgres/Redis/RabbitMQ.
- Worker/lock behavior is meaningfully different when Redis-backed locks are enabled in Compose.
