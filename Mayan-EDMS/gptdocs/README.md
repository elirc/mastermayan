# Mayan EDMS codebase-to-curriculum lab

This curriculum is for a junior full-stack engineer who knows CRUD applications and wants mid-level judgment: trace unfamiliar systems, protect contracts, test boundaries, and explain tradeoffs in interviews. Mayan EDMS is a modular Django document-management system whose core value is storing documents while preserving business context. It exposes server-rendered UI and a REST API, persists relational metadata plus binary files, and moves expensive work to Celery. It adds OCR, search, workflows, events, storage abstraction, and object-level ACLs. The repository is a single deployable Python application split into many Django apps, not a JavaScript monorepo. That difference is useful: map Django views/serializers/models/tasks to controllers/schema/domain/jobs in Node, and map templates to server-rendered UI.

Start with [the fast track](00-fast-track.md). Then choose a path:

| Learner | Route |
| --- | --- |
| Brand-new junior | cartography → stack → reading gym → testing |
| Familiar with Django/server frameworks | key flows → architecture → contribution practice |
| Mid-level new to Mayan | system map → pattern catalog → architecture critique |
| Senior reviewer | risks → architecture critique → design katas |
| Interview in two weeks | [two-week cram plan](08-interview-prep/07-two-week-cram-plan.md) → question banks → system design |

Use it over one weekend with `00-fast-track.md`, over two weeks with the cram plan, over eight weeks by completing one numbered module per week, or continuously by pairing tickets with STAR worksheets. Every real-code claim should link to exact lines. Fake examples begin with `# Illustrative fake code: not from this repo`. Drills include Basic/Solid/Strong self-grading. Unverified commands and hypotheses are labeled; see the [verification log](09-reference/verification-log.md).

The learning tracks are codebase navigation, Python/Django runtime fluency, architecture and reliability, quality/security, contribution, and interview communication. Keep the mindset ladder visible: a junior asks “how do I make it work?”; a mid-level engineer asks “is this the right pattern?”; a senior asks “what does this commit us to, who pays the cost, and how do we reduce risk?” Mid-level interviews test the second question and increasingly the third.

## Other curricula in this repo

This repository holds three independently written AI curricula for the same Mayan EDMS v4.3.1 snapshot. They do not link to each other, so pick one as your spine:

- `fabledocs/upskill/` — 53 files, aimed at a JS/TS engineer moving to Python/Django; includes the `08-interview-prep` module and a detailed verification log.
- `docs/upskill/` — 46 files, written for juniors through seniors without assuming a JS background; interview material is a single chapter (`07-career-and-collaboration/04-interview-prep-from-this-repo.md`).
- `gptdocs/` — 53 files with the same module layout as `fabledocs/upskill/`.

All three cite real file and line ranges; their link checks were rerun on 2026-10-06.
