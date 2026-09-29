# Writing PRs and RFCs

## PR description template (tuned to this repo)

```
## What
One sentence: the observable change.

## Why
The problem, with evidence (issue link, log, or anchor like
mayan/apps/ocr/tasks.py#L114-L131).

## How
Approach in 2-3 sentences. Which layer(s) touched. Which existing
pattern followed.

## Testing
- [ ] New/updated test: <name> (authz pair if it's an endpoint)
- [ ] Ran: make test MODULE=mayan.apps.<app> (verified/inferred)
- [ ] Manual check: <what you clicked/curled>

## Risk & blast radius
What could break; which apps/flows affected; is it additive?

## Rollback
How to undo (revert / flag off / reverse migration).

## Follow-ups
Anything intentionally out of scope (link an issue).

Signed-off-by: Real Name <email>   # DCO, via git commit -s
```

Why these sections: maintainers here are few and the codebase is big ([CONTRIBUTING.md](../../../CONTRIBUTING.md)) — a description that pre-answers "does it break anything, is it tested, how do I undo it" saves the review round-trip that kills momentum. The **Risk & blast radius** section is what separates a mid-level PR from a junior one; it shows you thought past "it works."

## Commit messages

Imperative mood, scoped, signed. House commits are terse; match them:

```
documents: fix unreachable OperationalError handler in OCR finisher

The general `except Exception` preceded the specific handler, making
retry-on-OperationalError unreachable. Reorder so transient DB errors
retry and unexpected errors record to the version error log.

Signed-off-by: Real Name <email>
```

`git commit -s` appends the sign-off automatically. Real name required — no pseudonyms ([CONTRIBUTING.md](../../../CONTRIBUTING.md)). Target the correct branch (`development` for new work, not `master`).

## When to write an RFC vs just a PR

- **Just PR:** bug fixes, tests, docs, additive fields, anything reversible and single-app (most of [06/01](../06-contribution-practice/01-good-first-tickets.md)).
- **RFC first:** behavior changes to documented/tested contracts (change-type authz, [06/M2](../06-contribution-practice/02-mid-level-feature-tickets.md)), new cross-cutting subsystems (dead-letter, async fan-out), schema changes with data migration, anything touching the authorization or event core. If you'd have to update an existing test's *assertions* (not just add tests), you probably need an RFC.

## RFC template

```
# RFC: <title>

## Problem
What's wrong, with evidence (anchors, benchmarks, incident).

## Constraints / invariants that must hold
e.g. "audit event stays atomic with the mutation";
"in-flight queue messages must remain valid across deploy".

## Options
### Option A — <name>
Sketch, pros, cons, blast radius.
### Option B — <name>
...

## Recommendation
Which and why. What you deliberately reject.

## Migration path
Deploy order; two-release plan for renames; data backfill;
in-flight message compatibility.

## Test plan
What proves it works; what pins the old behavior first.

## Rollback
Reversible? Flag? Reverse migration?

## Open questions
The honest unknowns.
```

The [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) and every senior project ([06/03](../06-contribution-practice/03-senior-build-projects.md)) are written in this voice — reuse them as worked examples.

## What gets a PR rejected here (learn the failure modes)

- No test (hard rule).
- No DCO sign-off / pseudonym.
- Targets `master` instead of `development`.
- Drive-by refactor mixed with the fix (keep PRs single-purpose).
- Behavior change to a tested contract with no discussion.
- Breaks house style (a maintainer will not hand-fix your quotes).
- Changes task kwargs non-additively (breaks in-flight messages).

**Drill:** write the full PR description for ticket 1 ([06/01](../06-contribution-practice/01-good-first-tickets.md)). Then write the RFC opening (Problem + Constraints + two Options) for ticket M2. *Strong:* your M2 RFC's Constraints section names "the existing test enshrines the loose behavior" as an invariant to *deliberately change*, with the release-note plan — proving you see the process, not just the code.

**Interview angle:** "How do you communicate a risky change?" → the RFC template. "Walk me through a PR you're proud of." → use the description template as the spine of the story ([08/06](../08-interview-prep/06-behavioral-star-stories.md)).
