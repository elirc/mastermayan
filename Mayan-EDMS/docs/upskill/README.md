# Upskill Curriculum for Mayan EDMS

This curriculum turns this clone of Mayan EDMS into a deliberate training lab. It is for juniors who want structured growth, mid-level engineers who need a repo-specific ownership path, and seniors who want a fast architecture and risk review without losing touch with the real code.

Mayan EDMS is a large Django application organized as many internal apps under `mayan/apps`, with Docker-first deployment, Celery workers, DRF APIs, ACL-driven authorization, and document-processing pipelines such as upload, parsing, OCR, indexing, and metadata extraction. The root settings wire together a single runtime surface with many domain modules in [`mayan/settings/base.py`](../../mayan/settings/base.py#L34-L123), while URL and API surfaces are spread across app-local `urls.py` files such as [`mayan/apps/documents/urls.py`](../../mayan/apps/documents/urls.py#L95-L688) and [`mayan/apps/rest_api/urls.py`](../../mayan/apps/rest_api/urls.py#L10-L45). The repo is not a tiny tutorial app: it contains operational concerns, upgrade paths, backward-compatibility constraints, and worker-specific failure handling that are worth studying directly.

## Who this is for

- Brand-new junior engineer: learn how to navigate a serious Django codebase without guessing.
- Junior with basic Python/Django familiarity: practice making safe changes that follow existing boundaries.
- Mid-level engineer new to the repo: map cross-layer flows, tests, and risks quickly.
- Senior engineer: review contracts, async behavior, authorization, and migration pressure points.

## How to use it

### One weekend

1. Read [00-fast-track.md](./00-fast-track.md).
2. Skim [01-system-map.md](./01-codebase-cartography/01-system-map.md) and [05-key-flows.md](./01-codebase-cartography/05-key-flows.md).
3. Do one annotation drill and one debugging scenario.
4. End by attempting one ticket from [01-good-first-tickets.md](./06-contribution-practice/01-good-first-tickets.md).

### Two weeks

1. Finish the cartography module.
2. Work through stack/runtime and architecture modules.
3. Do at least four drills, two review katas, and two test recipes.
4. Draft design notes for one mid-level feature ticket.

### Eight weeks

1. Rotate through one representative flow per week: upload, ACL/API, metadata, parsing, OCR, checkouts, indexing, auth.
2. Pair each flow with one transferable theme: validation, idempotency, caching, permissions, or performance.
3. Ship 3-5 small contributions and one cross-layer change.
4. Use the interview prep file as a recurring teach-back exercise.

### Ongoing contribution practice

- Keep [command-cheatsheet.md](./08-reference/command-cheatsheet.md) open while working.
- Before each change, read the matching pattern card and testing recipe.
- After each change, log what surprised you in [verification-log.md](./08-reference/verification-log.md).

## Learning tracks

- `01-codebase-cartography`: repo shape, file order, domain vocabulary, major flows.
- `02-stack-and-language-mastery`: Python, Django, DRF, Celery, and the repo’s JavaScript surfaces through real files.
- `03-architecture-and-patterns`: boundaries, persistence, auth, async, pattern recognition, critique.
- `04-code-reading-gym`: drills, trace tables, fake-code contrasts, review practice.
- `05-quality-engineering`: testing, debugging, security, performance, observability.
- `06-contribution-practice`: realistic tickets, projects, and design exercises.
- `07-career-and-collaboration`: PRs, RFCs, review language, maintainer communication, interview prep.
- `08-reference`: commands, risks, rubrics, and verification notes.

## Recommended paths

- Brand-new junior:
  Start with [00-fast-track.md](./00-fast-track.md), then [02-file-reading-order.md](./01-codebase-cartography/02-file-reading-order.md), then [01-annotation-drills.md](./04-code-reading-gym/01-annotation-drills.md).
- Junior with basic stack familiarity:
  Add [01-language-runtime-model.md](./02-stack-and-language-mastery/01-language-runtime-model.md), [03-validation-auth-and-permissions.md](./03-architecture-and-patterns/03-validation-auth-and-permissions.md), and [02-writing-tests-here.md](./05-quality-engineering/02-writing-tests-here.md).
- Mid-level engineer new to this repo:
  Jump from [01-system-map.md](./01-codebase-cartography/01-system-map.md) to [05-key-flows.md](./01-codebase-cartography/05-key-flows.md), then [05-pattern-catalog.md](./03-architecture-and-patterns/05-pattern-catalog.md) and [02-mid-level-feature-tickets.md](./06-contribution-practice/02-mid-level-feature-tickets.md).
- Senior engineer doing architecture review:
  Read [01-system-map.md](./01-codebase-cartography/01-system-map.md), [06-architecture-critique.md](./03-architecture-and-patterns/06-architecture-critique.md), [risk-register.md](./08-reference/risk-register.md), and [04-interview-prep-from-this-repo.md](./07-career-and-collaboration/04-interview-prep-from-this-repo.md).

## Conventions

- File anchors:
  Important claims cite repo files with links or `path:line-line`, for example [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L138-L176).
- Fake code labeling:
  Illustrative snippets are marked with `// Illustrative fake code: not from this repo.` or the Python equivalent.
- Drills:
  Most pages ask you to trace, predict, annotate, or design instead of just reading.
- Self-grading:
  Every drill distinguishes weak, solid, and strong answers.
- Verification notes:
  Key pages include what was inspected, which commands were run, and what remains uncertain.

## Senior mindset

Junior asks: how do I make it work? Mid-level asks: is this the right pattern for this codebase? Senior asks: what does this change commit us to, who pays the cost, what is the rollback, and how do we reduce blast radius before production learns the answer for us?

## Verification Notes

- Inspected root metadata: [`README.md`](../../README.md), [`CONTRIBUTING.md`](../../CONTRIBUTING.md), [`Makefile`](../../Makefile), [`tox.ini`](../../tox.ini), [`.gitlab-ci.yml`](../../.gitlab-ci.yml).
- Inspected runtime wiring: [`mayan/settings/base.py`](../../mayan/settings/base.py), [`mayan/celery.py`](../../mayan/celery.py), [`docker/docker-compose.yml`](../../docker/docker-compose.yml).
- Ran: `git status --short`, `python --version`, `pip --version`.
- Attempted and failed due missing dependency: `python manage.py check --settings=mayan.settings.testing.development` because Django is not installed in the current environment.
