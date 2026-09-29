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
