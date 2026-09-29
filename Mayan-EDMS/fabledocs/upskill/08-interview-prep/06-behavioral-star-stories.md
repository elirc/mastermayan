# Behavioral STAR stories

Ten worksheets. Behavioral rounds are scored on *specificity* and *reflection*, not heroics. Since you may not have shipped these changes, source them from **studying this repo** — "walk me through how you learned a complex codebase" is a legitimate, strong story, and completing any ticket in [06](../06-contribution-practice/README.md) gives you a real one. Fill each S/T/A/R in your own words; the scaffolding is here.

Format: Situation → Task → Action → Result, plus evidence to cite, senior-signal detail, and a one-line resume bullet.

---

## Story 1: "Tell me about a complex system you had to understand quickly"
**Source:** studying Mayan (this curriculum).
**S:** inherited/joined a 2,181-file, 57-app Django document system with no prior context. **T:** become productive enough to trace flows and propose changes. **A:** found the per-app skeleton and the registration idioms; traced two end-to-end flows (upload, authorization) using file:line anchors; built a mental model of the boundaries before touching code. **R:** could locate any feature and explain the authz path within days.
**Senior signal:** you learned the *shape* (registry idiom, worker tiers) not just individual files.
**Bullet:** "Ramped on a 57-module legacy Django codebase by mapping its registration architecture and tracing critical flows end to end."

## Story 2: "A time you found a subtle bug others missed"
**Source:** the OCR unreachable-handler or `ignore_results` typo ([06/01 tickets 1–2](../06-contribution-practice/01-good-first-tickets.md)).
**S/T:** reviewing async task code. **A:** noticed exception-ordering made a handler dead code / a config kwarg silently no-op; verified the behavior; fixed with a regression test. **R:** removed a latent failure and documented the exception-MRO gotcha for the team.
**Senior signal:** you added the test that prevents recurrence, not just the fix.
**Bullet:** "Identified and fixed silent error-handling defects (unreachable handlers, no-op config) with regression coverage."

## Story 3: "A technical tradeoff you weighed"
**Source:** the change-type authz asymmetry ([03/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)).
**S:** found that edit-permission let users move documents into any type (with different retention). **T:** decide whether/how to tighten it. **A:** weighed security vs a test-enshrined existing contract; proposed a setting-gated change with a changelog and deprecation path rather than a silent break. **R:** a safe rollout plan for a behavior change.
**Senior signal:** you treated it as a rollout problem, not just a code fix.
**Bullet:** "Proposed a backward-compatible authorization tightening with a phased rollout to avoid breaking existing deployments."

## Story 4: "A time you disagreed with someone"
**Source:** review kata 4 / the `is_stub` debate ([04/04](../04-code-reading-gym/04-review-katas.md)).
**S:** a colleague wanted to flip an "obviously inverted" flag. **T:** prevent a data-loss bug without being obstructive. **A:** showed with evidence that the flag was load-bearing (the reaper would auto-delete documents), asked for a test capturing intended behavior, offered to pair. **R:** avoided a destructive change; the real decision got made deliberately.
**Senior signal:** evidence + a collaborative path, not authority.
**Bullet:** "Blocked a well-intentioned change that would have caused data loss, using evidence and a proposed test to reach a shared decision."

## Story 5: "A time you improved reliability/observability"
**Source:** the dead-letter/observability project ([06/03 project 1](../06-contribution-practice/03-senior-build-projects.md)).
**S:** background task failures vanished into logs with no failure rate or retry. **T:** make failures visible and recoverable. **A:** designed a `FailedTask` capture at the Celery signal layer + admin retry + metrics. **R:** operators could see and re-drive failures instead of fielding blind support tickets.
**Senior signal:** you captured at the one integration point that catches all tasks.
**Bullet:** "Designed a dead-letter and failure-visibility subsystem for a queue-based pipeline."

## Story 6: "A time you improved test coverage meaningfully"
**Source:** the authz test-pair gap / version-modification tests ([05/01](../05-quality-engineering/01-testing-strategy.md)).
**S:** an async endpoint had no direct test and a suspect query path. **T:** pin behavior and catch the defect. **A:** added a model-level test invoking the method directly (no Celery), which surfaced the wrong `order_by`. **R:** turned a silent runtime failure into a caught-at-CI one.
**Senior signal:** you tested at the layer that isolates the bug.
**Bullet:** "Added targeted regression tests that surfaced a latent query defect in an untested async path."

## Story 7: "A time you had to learn something outside your stack"
**Source:** you (JS/TS) reading a Python/Django/Celery codebase (this whole curriculum).
**S:** strong in React/Node, needed to contribute to Django. **T:** map unfamiliar idioms to known ones fast. **A:** built the translation table (managers ≈ query scopes, Celery ≈ BullMQ, signals ≈ EventEmitter, decorators ≈ HOCs); anchored each to real files. **R:** contributed without a multi-week ramp.
**Senior signal:** you learned by *mapping*, not memorizing.
**Bullet:** "Cross-trained from a JS/TS background to contribute to a Python/Django/Celery codebase by systematically mapping concepts."

## Story 8: "A time you dealt with ambiguity"
**Source:** the `is_stub` semantics investigation ([06/01 ticket 3](../06-contribution-practice/01-good-first-tickets.md)).
**S:** code contradicted its own documentation; unclear which was intended. **T:** resolve without guessing. **A:** traced the reaper interaction to understand the *consequence* of each interpretation, wrote a test pinning current behavior, and escalated the decision with the tradeoff laid out. **R:** an informed decision instead of a coin-flip change.
**Senior signal:** you made the ambiguity *decidable* by surfacing the consequence.
**Bullet:** "Resolved ambiguous, self-contradicting behavior by tracing downstream consequences and escalating a clear decision."

## Story 9: "A time you pushed back on scope / said no"
**Source:** the watch-folder batch or archive-expand tickets ([06/01 tickets 15, 19](../06-contribution-practice/01-good-first-tickets.md)).
**S:** a "quick" change (ingest all files per tick) had a hidden distributed-lock hazard. **T:** avoid shipping a concurrency bug. **A:** flagged that it needed a design discussion (lock-budget, batch bounds) before implementation; filed an issue instead of a rushed PR. **R:** the change shipped later, safely.
**Senior signal:** you recognized "this needs an RFC" — a promotion-worthy instinct.
**Bullet:** "Recognized a deceptively-simple task as a distributed-systems risk and drove it through proper design review."

## Story 10: "A mistake you made and what you learned"
**Source:** adapt honestly — e.g. an early assumption while learning (thinking a new upload replaces the file, then discovering the file/version model).
**S:** assumed uploads mutate the document's bytes. **T:** correct my mental model before building on it. **A:** traced `file_new` and the version-remap path, corrected the model, updated my notes. **R:** avoided designing a feature on a false assumption.
**Senior signal:** you caught the wrong model *before* it cost code.
**Bullet:** "Course-corrected an incorrect data-model assumption early by tracing the code rather than trusting intuition."

---

## Mapping to common prompts

| Prompt | Best stories |
| --- | --- |
| Conflict | 4, 9 |
| Ambiguity | 8, 10 |
| Mistake | 10, 8 |
| Technical tradeoff | 3, 5 |
| Leadership/initiative | 5, 9 |
| Learning fast | 1, 7 |

**Rehearsal check for every story:** under 2 minutes? starts with concrete Situation (not backstory)? ends with a measurable/observable Result? has one senior-signal detail (a tradeoff weighed, a risk reduced, a person helped)? If any is no, rewrite. Record stories 1, 3, and 4 — they cover the most prompts.
