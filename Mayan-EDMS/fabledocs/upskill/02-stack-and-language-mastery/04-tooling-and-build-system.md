# Tooling and build system

## Dependency management: pip + pinned tiers (your package-lock, done manually)

No lockfile ecosystem here: [requirements.txt](../../../requirements.txt) includes tiered files; [requirements/base.txt](../../../requirements/base.txt) pins **exact versions** (`celery==5.2.3`, `djangorestframework==3.13.1`, `Pillow==9.2.0`…). Exact pins = reproducible builds (the lockfile's job) at the cost of manual upgrades (no `npm audit fix`). Dev/test/build extras live in sibling files under `requirements/`.

Notable: the app also manages **frontend and binary dependencies itself** — the `dependencies` app declares JS libs with versions and hashes ([appearance/dependencies.py](../../../mayan/apps/appearance/dependencies.py#L38-L51)), downloaded at install time. That is "vendored npm" implemented in Django — unusual, and worth mentioning in an interview as a deliberate offline/self-hosted-friendly choice.

## The build artifact is a Docker image

CI (GitLab) runs: test → build python package → **build docker image → run the test suite inside the image** → push ([.gitlab-ci.yml](../../../.gitlab-ci.yml#L1-L50), see the `docker run … run_tests` line at [L43](../../../.gitlab-ci.yml#L36-L45)). Treat that as the real definition of "works": the deliverable is the image, and tests gate it *in situ*. Compose wires runtime deps ([docker/docker-compose.yml](../../../docker/docker-compose.yml#L3-L16)).

## Migrations are build outputs you commit

Each app owns its `migrations/` chain (e.g. [documents/migrations/](../../../mayan/apps/documents/migrations/) — 80+ files). House habits: tests skip migrations by default for speed (`--skip-migrations` via [Makefile](../../../Makefile#L8-L16)) but there are **dedicated migration tests** (`test_migrations.py` in several apps, run with `MayanMigratorTestCase`, [testing/tests/base.py](../../../mayan/apps/testing/tests/base.py#L83-L88)). Testing a migration is rare discipline; steal it. Rules for changing schema here → [03/02-data-model-and-persistence.md](../03-architecture-and-patterns/02-data-model-and-persistence.md).

## Task running: Makefile as the single entry point

[Makefile](../../../Makefile#L1-L60) wraps everything: `make test MODULE=…`, `make test-all`, DB-specific runs against dockerized Postgres/MySQL/Oracle (container names at [L18–L22](../../../Makefile#L18-L22)), plus release chores. When you join any repo, find this layer first — package.json scripts, justfile, Makefile — because it encodes *how the maintainers actually work*, which beats the README when they disagree.

## Settings pipeline (build-time vs run-time config)

`mayan.settings.production` is the default (`DJANGO_SETTINGS_MODULE` set in [mayan/celery.py](../../../mayan/celery.py#L7)); variants exist for development/staging/testing ([mayan/settings/](../../../mayan/settings/)). Runtime config comes from `MAYAN_*` env vars processed by the smart-settings singleton ([settings/base.py](../../../mayan/settings/base.py#L18-L28)). Secret key: env var, else a file under the media root, else a default placeholder ([settings/base.py](../../../mayan/settings/base.py#L30-L38)) — trace that fallback chain once; "where does the secret come from" is a classic ops interview probe.

## What's missing (say it, don't hide it)

No enforced formatter/linter config in-repo, no type checking, no pre-commit hooks, no JS build step (libraries are vendored, not bundled). For a 2022 self-hosted Python project this is normal; for *you* it means style discipline is social, not mechanical. When you port these ideas to a TS project, the equivalents are: exact pins → lockfile; Makefile → package.json scripts; migration tests → your ORM's migration CI step; smart settings → typed env parsing (envalid/zod).

## Interview angle

1. *"Walk me through your CI pipeline"* → describe this one honestly: test, package, image build, in-image tests, push; then contrast with what you'd add (lint gate, SAST, image scan).
2. *"How do you keep dev/prod parity?"* → the image-is-the-artifact argument.
3. *"How do you test migrations?"* → `MayanMigratorTestCase` exists; most shops have nothing. Memorable answer material.

Cards: [08/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q12, [08/04 system design](../08-interview-prep/04-system-design-from-this-repo.md) scaling section.

**Drill:** using only the Makefile and CI file, write the exact command sequence a maintainer runs to test the `documents` app against MySQL. Mark each command `verified`/`inferred`. *Strong:* you notice tests default to `mayan.settings.testing.development` ([Makefile L12–L14](../../../Makefile#L12-L16)) and can say why a distinct testing settings module exists (in-memory/faster backends, deterministic storage paths).
