# Mayan EDMS — codebase-to-curriculum training lab

A learning suite that turns this repository into a training lab for a **junior fullstack JS engineer** (React/Node/TS CRUD background) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles. Every page teaches two things at once: how *this* codebase works (with exact file/line anchors) and the transferable pattern you can carry to any repo and any interview.

## What this repo is

Mayan EDMS v4.3.1 ([version constants](../../mayan/__init__.py#L1-L12)) is a mature, self-hosted **electronic document management system**: upload documents, OCR them, tag them, index them, route them through approval workflows, and control who sees what down to the individual object. It is a single deployable **Django 3.2** application split into **57 pluggable Django apps** under [mayan/apps/](../../mayan/apps/), not a JS monorepo — which is exactly why it is a good gym: you will map serializers to your DTOs, Celery workers to your job queues, and Django templates to server-rendered UI, and in doing so learn the ideas rather than the framework incantations. Its three signature subsystems are: object-level **ACLs** enforced by queryset filtering ([acls/managers.py](../../mayan/apps/acls/managers.py#L268-L294)), an **event system** driven by decorators on model methods ([events/decorators.py](../../mayan/apps/events/decorators.py#L8-L33)), and a four-tier **Celery worker topology** where every slow operation is a queued task ([task_manager/workers.py](../../mayan/apps/task_manager/workers.py#L12-L36)). Storage is pluggable (filesystem/S3-style), search is pluggable (Whoosh/Elasticsearch), and even upload sources (web form, watch folder, email inbox, scanner) are plugin backends.

## How to use this curriculum

| Time budget | Path |
| --- | --- |
| One weekend | [00-fast-track.md](00-fast-track.md) end to end |
| Two weeks (interview!) | [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md) |
| Eight weeks | One numbered module per week; do every drill; modules 04 and 06 are where the learning actually happens |
| Ongoing contribution | [06-contribution-practice/](06-contribution-practice/README.md) tickets, paired with [08-interview-prep/06-behavioral-star-stories.md](08-interview-prep/06-behavioral-star-stories.md) |

Recommended reading order per learner:

- **Brand-new junior**: 00 → 01 (all) → 02 → 04 (drills) → 05-quality → come back for 03.
- **Junior who knows Python/Django basics**: 01-cartography/05-key-flows → 03-architecture → 04-reading-gym → 06-contribution.
- **Mid-level, new to this repo**: 01-system-map → 03-pattern-catalog → 03-architecture-critique → 06-mid-level tickets.
- **Senior doing an architecture review**: [09-reference/risk-register.md](09-reference/risk-register.md) → [03-architecture-and-patterns/06-architecture-critique.md](03-architecture-and-patterns/06-architecture-critique.md) → 06-contribution-practice/04-refactor-and-design-katas.md.
- **Candidate with an interview in two weeks**: go straight to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md); it schedules everything else.

## Conventions used everywhere

- **File anchors**: every claim about real code links to a path with line numbers, e.g. [document_models.py](../../mayan/apps/documents/models/document_models.py#L142-L163). Line numbers were confirmed against this working copy (v4.3.1) at authoring time; if a link looks stale, trust the symbol name.
- **Fake code is labeled**: every invented snippet starts with `# Illustrative fake code: not from this repo`. Anything unlabeled is real.
- **Verification labels**: commands are marked `verified` (actually run) or `inferred` (read from Makefile/CI/docs but not executed — the norm here, since this Django 3.2 app can't run on the authoring machine's Python 3.13). Bug-shaped observations are labeled *investigate* or *possible risk*, never asserted as bugs. See the [verification log](09-reference/verification-log.md).
- **Drills with self-grading**: most sections end with a task and a Basic/Solid/Strong rubric. Grade yourself honestly; "Strong" is the mid-level interview bar.
- **Interview angle**: sections flag how the material shows up in interviews and cross-link into [08-interview-prep/](08-interview-prep/README.md).

## Other curricula in this repo

This repository holds three independently written AI curricula for the same Mayan EDMS v4.3.1 snapshot. They do not link to each other, so pick one as your spine:

- `fabledocs/upskill/` — 53 files, aimed at a JS/TS engineer moving to Python/Django; includes the `08-interview-prep` module and a detailed verification log.
- `docs/upskill/` — 46 files, written for juniors through seniors without assuming a JS background; interview material is a single chapter (`07-career-and-collaboration/04-interview-prep-from-this-repo.md`).
- `gptdocs/` — 53 files with the same module layout as `fabledocs/upskill/`.

All three cite real file and line ranges; their link checks were rerun on 2026-10-06.

## The learning tracks

1. **Cartography** (01) — find anything in 2,181 Python files without reading them all.
2. **Stack mastery** (02) — Python/Django/Celery mental models, mapped from the JS/TS concepts you already have.
3. **Architecture** (03) — boundaries, persistence, authorization, async reliability, and a pattern catalog.
4. **Reading gym** (04) — annotation drills, trace tables, bad-vs-better contrasts, review katas.
5. **Quality engineering** (05) — testing strategy, debugging method, performance, security, operations.
6. **Contribution practice** (06) — 20 junior tickets, 12 mid-level tickets, 6 senior projects, design katas.
7. **Career & collaboration** (07) — reviews, PRs, RFCs, maintainer communication.
8. **Interview prep** (08) — 50+ question cards (most anchored to this repo), a system-design walkthrough, timed rounds, STAR stories, and a two-week cram plan.

## The mindset ladder

- A **junior** asks: *how do I make it work?*
- A **mid-level** engineer asks: *is this the right pattern? What breaks it?*
- A **senior** asks: *what does this commit us to, who pays the cost, and how do we reduce the risk?*

Interviews for mid-level roles test the second question relentlessly and probe for the third. When this curriculum asks you to critique `AccessControlListManager` or design a retry policy, it is rehearsing exactly that. Answer every drill out loud at least once — the skill being hired is *articulated* judgment, not silent judgment.
