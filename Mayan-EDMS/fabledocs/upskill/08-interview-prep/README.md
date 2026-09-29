# 08 — Interview prep

First-class module. Target: **mid-level fullstack JS interviews**. The trick you're practicing: most candidates answer with definitions; you'll answer with a *concrete example, a tradeoff, and a failure mode* — drawn from real code you can describe. Over 60% of the question cards here are anchored to Mayan EDMS, so you rehearse with evidence, not trivia. (Mayan is Python/Django, but the *concepts* — async work, authz, data modeling, caching, testing — are language-agnostic; each card includes the JS/TS translation.)

| File | Round | Cards |
| --- | --- | --- |
| [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) | Language/runtime | 14 |
| [02-frontend-framework-questions.md](02-frontend-framework-questions.md) | Frontend | 11 |
| [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) | API/data | 18 |
| [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) | System design | 1 full walkthrough + 4 variations |
| [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) | Practical | 4 debug + 3 review, timed |
| [06-behavioral-star-stories.md](06-behavioral-star-stories.md) | Behavioral | 10 STAR worksheets |
| [07-two-week-cram-plan.md](07-two-week-cram-plan.md) | Plan | day-by-day |

## How mid-level loops are usually structured

1. **Recruiter/tech screen** — high-level, "tell me about a project," one or two concept checks.
2. **Technical deep-dive** — language/framework internals; they follow up until you hit your edge (that's the point — find it before the interview).
3. **Practical coding** — implement or debug something small; think aloud.
4. **System design** — "design X"; for mid-level they want a reasonable data model + API + one scaling insight, not a distributed-systems dissertation.
5. **Behavioral** — conflict, ambiguity, mistakes, tradeoffs.

## The golden rule

Every answer = **definition (one line) → concrete example (with a name/anchor) → tradeoff → failure mode.** "Idempotency means safe to repeat. In Mayan, per-page OCR upserts content keyed by page so a retry is harmless; the *upload* task isn't idempotent, so a client retry can create a duplicate document — which is why real systems add an idempotency key." That answer, in 20 seconds, signals mid-level. A bare definition signals junior. Practice the arc until it's automatic.

## Card format

Each card: the question as asked → what it's really testing → repo anchor → junior/mid/senior answer contrast → likely follow-ups → a practice drill. Read the anchor **before** rehearsing so your example is concrete.
