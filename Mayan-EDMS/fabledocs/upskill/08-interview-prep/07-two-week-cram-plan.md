# Two-week cram plan

Assumes ~2–3 focused hours/day. Adjust, but keep the **teach-back and record-yourself** habits — passive reading is the trap. Checkpoints at day 7 and day 14. Everything links back into the curriculum.

## Week 1 — build the foundation and the examples

**Day 1 — Orientation.** Read [README](../README.md), [00-fast-track](../00-fast-track.md), [01/01-system-map](../01-codebase-cartography/01-system-map.md). Get the app or its tests running (Docker). Teach-back: describe Mayan's architecture in 5 sentences, out loud.

**Day 2 — The two flows that anchor everything.** Study [Flow 1 (upload)](../01-codebase-cartography/05-key-flows.md) and [Flow 3 (authorization)](../01-codebase-cartography/05-key-flows.md) until you can trace each without notes. These power half your interview answers.

**Day 3 — Stack translation.** [02/01 runtime](../02-stack-and-language-mastery/01-language-runtime-model.md) + [02/02 frameworks](../02-stack-and-language-mastery/02-framework-mental-models.md). Do the drills. Rehearse [08/01 cards Q1, Q7](01-js-ts-node-deep-dive.md) (event loop, idempotency) out loud.

**Day 4 — Data and authz.** [03/02 data model](../03-architecture-and-patterns/02-data-model-and-persistence.md) + [03/03 authz](../03-architecture-and-patterns/03-validation-auth-and-permissions.md). Master the change-type asymmetry ([08/03 Q8](03-api-and-data-modeling-questions.md)) — your strongest single answer.

**Day 5 — Async & reliability.** [03/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md) + Flows 5, 7. Rehearse [08/03 Q9, Q10](03-api-and-data-modeling-questions.md) (transactions, search consistency).

**Day 6 — Patterns + first timed debug.** Skim [pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md); do [debug round 1](05-debugging-and-code-review-rounds.md) with a 15-min timer, recorded.

**Day 7 — CHECKPOINT + rest.** Self-assess (below). Do [annotation drills 1–5](../04-code-reading-gym/01-annotation-drills.md). Light day.

### Day 7 self-assessment
- [ ] Trace upload and authz flows from memory, out loud, under 3 min each.
- [ ] Explain event loop vs process concurrency with the worker-tier example.
- [ ] State the change-type authz bug with anchors.
- [ ] Deliver the idempotency answer in the full arc (definition→example→tradeoff→failure).
If any box is unchecked, that's Monday's priority.

## Week 2 — interview reps

**Day 8 — API/data card bank.** Drill all of [08/03](03-api-and-data-modeling-questions.md); target Q6, Q8, Q9, Q10, Q15 cold. Record 3.

**Day 9 — System design.** Full [08/04 walkthrough](04-system-design-from-this-repo.md) on a whiteboard, 35 min, out loud, recorded. Then variation 1 (multi-tenant).

**Day 10 — System design again + JS/TS.** Redo the walkthrough hitting the junior/mid/senior ladder at each step; variation 3 (10x traffic). Drill remaining [08/01 cards](01-js-ts-node-deep-dive.md).

**Day 11 — Debugging rounds.** Timed [debug rounds 2 and 4](05-debugging-and-code-review-rounds.md), recorded. Listen back for "say the check before checking."

**Day 12 — Code-review rounds.** Timed [review rounds 1 and 3](05-debugging-and-code-review-rounds.md). Practice the review voice ([07/01](../07-career-and-collaboration/01-code-review-mindset.md)): question, quantify, cite precedent.

**Day 13 — Behavioral.** Fill and rehearse [STAR stories 1, 3, 4, 5, 8](06-behavioral-star-stories.md). Under 2 min each, recorded. These cover conflict/ambiguity/tradeoff/reliability.

**Day 14 — CHECKPOINT + mock loop.** Simulate a full loop: 1 system-design (35 min) + 1 debug (15) + 1 review (15) + 2 behavioral (10). Ideally with a friend; else record and self-grade against the rubrics.

### Day 14 self-assessment (the interview-ready bar)
- [ ] System design: full walkthrough + one variation, ladder at each step, 6+ anchors from memory.
- [ ] Any [08/03](03-api-and-data-modeling-questions.md) card in the arc, two follow-ups deep, for Q6/Q8/Q9/Q10.
- [ ] A debugging round narrating hypotheses-before-probes, ending with a regression test.
- [ ] A review round separating code from process, refusing ambiguous approvals.
- [ ] Five STAR stories under 2 min with a senior-signal detail each.

## If you have only one week
Days 1, 2, 4 (authz), 5 (async), 9 (system design), 13 (behavioral), 14 (mock). Skip the frontend cards ([08/02](02-frontend-framework-questions.md)) unless the role is frontend-heavy — but keep the "why server-rendered" argument (Q1) in your pocket.

## If you have only 48 hours
[00-fast-track](../00-fast-track.md) + Flows 1 & 3 + [08/04 walkthrough](04-system-design-from-this-repo.md) + [08/03 Q6, Q8, Q9](03-api-and-data-modeling-questions.md) + STAR stories 1 & 3. Rehearse each out loud twice. That's a credible mid-level showing.

**The one habit that matters:** every answer, every day, out loud in the arc. Interviews reward *articulated* judgment. Silent understanding scores zero.
