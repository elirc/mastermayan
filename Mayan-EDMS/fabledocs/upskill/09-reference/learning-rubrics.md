# Learning rubrics

Observable behaviors, not vague traits. Self-assess honestly per skill; the "Interview-ready" column is the mid-level hiring bar. Rate yourself Junior / Mid / Senior by what you can *do*, not what you've read.

## Codebase navigation

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Find a feature | greps by keyword | predicts the app + file from the skeleton | knows the registration idioms that hide wiring | can explain how 57 apps compose ([system map](../01-codebase-cartography/01-system-map.md)) without notes |
| Trace a flow | follows one file | traces across layers with anchors | names invariants & side effects at each step | narrates upload + authz flows in <3 min ([Flows 1,3](../01-codebase-cartography/05-key-flows.md)) |

## Language/runtime fluency

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Concurrency | "async/threads" | process vs event-loop models | worker tiers, GIL scope, starvation | delivers Q1/Q12 with the worker-tier example ([08/01](../08-interview-prep/01-js-ts-node-deep-dive.md)) |
| Error handling | try/catch | exception ordering, narrow catches | retry taxonomy, silent-failure risks | cites the unreachable-handler bug + retry table |

## Architecture judgment

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Boundaries | names layers | spots a boundary leak | states the tradeoff both ways | volunteers a leak *with* its counterargument ([03/01](../03-architecture-and-patterns/01-boundaries-and-layers.md)) |
| Authorization | "check the user" | filter-at-queryset + 404 | inheritance, bypass populations, the change-type asymmetry | delivers [08/03 Q6/Q8](../08-interview-prep/03-api-and-data-modeling-questions.md) cold |
| Async reliability | "use a queue" | idempotency, at-least-once | reapers, dead-letters, lock timeouts | maps a repo example to each ([03/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)) |

## Quality engineering

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Testing | writes a happy-path test | authz pair + event assertions | picks the layer that isolates the bug; migration tests | explains "how do you test authz" with the pair convention |
| Debugging | guesses a fix | narrates hypotheses, cheap probes | branch-structure-first; code vs config | passes a timed round ending in a regression test ([08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md)) |
| Performance | "add an index" | measure-first, `assertNumQueries` | names the ACL hotspot, proposes+proves | delivers the method template ([05/04](../05-quality-engineering/04-performance-thinking.md)) |
| Security | "sanitize input" | IDOR-via-queryset, validation layers | related-write authz, archive bombs, sandboxing | delivers [08/03 Q6/Q18](../08-interview-prep/03-api-and-data-modeling-questions.md) with anchors |

## Contribution & collaboration

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| PRs | "it works" | tests + risk section | blast radius, rollback, two-release renames | writes the full PR template unprompted ([07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)) |
| Review | finds surface bugs | finds blocking + test gap | refuses ambiguous approvals; code vs process | passes a timed review round ([08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md)) |
| Judgment | refactors for prettiness | refactors to cut a named cost | sometimes says "no change, here's why" | argues [kata 5/8](../06-contribution-practice/04-refactor-and-design-katas.md) both ways |

## Interview communication

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Answer shape | definition only | definition + example | + tradeoff + failure mode | the full arc, automatically, on any [08/03](../08-interview-prep/03-api-and-data-modeling-questions.md) card |
| System design | draws boxes | boxes + one tradeoff | ladder at each step; measure-first; when the fancy choice is wrong | full [08/04 walkthrough](../08-interview-prep/04-system-design-from-this-repo.md) + a variation |
| Behavioral | vague heroics | STAR structure | senior-signal detail; reflection | 5 stories <2 min with measurable results ([08/06](../08-interview-prep/06-behavioral-star-stories.md)) |

## Self-assessment checklist (tick when true, out loud)

- [ ] I can trace the upload flow and the authorization flow from memory.
- [ ] I can name a real bug in this repo and the test that would catch it.
- [ ] I can give the change-type authz answer with file anchors.
- [ ] I can run the full system-design walkthrough with the junior/mid/senior ladder.
- [ ] I answer every concept in the arc: definition → example → tradeoff → failure mode.
- [ ] I have 5 STAR stories rehearsed under 2 minutes.
- [ ] I can review a diff by leading with a question, quantifying impact, and citing in-repo precedent.
- [ ] I know one thing in this codebase I'd *refuse* to change, and why.

Seven of eight ticked, out loud, is interview-ready for mid-level fullstack.
