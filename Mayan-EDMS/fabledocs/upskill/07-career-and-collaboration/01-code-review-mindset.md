# Code review mindset

## The five layers (review in this order)

1. **Does it work?** Run it / read the test. No test → the PR is incomplete here (house rule: [CONTRIBUTING.md](../../../CONTRIBUTING.md) requires a test per change).
2. **Is it correct?** Edge cases, error paths, the failure table. For this repo: what happens on the *async* half, on retry, on the denied user?
3. **Will it stay correct?** Does it protect an invariant or lean on a convention that the next person will break? (e.g. a new view that queries `Document.objects` instead of `valid` silently bypasses trash-hiding.)
4. **Does it fit?** House style (single quotes, keyword-args, alphabetized imports), the app skeleton, the test-pair convention, the queue contract.
5. **Is it kind to future maintainers?** Names, comments that explain *why*, blast radius stated in the description.

Most junior reviews stop at layer 1. Mid-level reviews live at 2–3. The senior differentiator is catching layer-3 issues — "this works today but couples X to Y" — which is exactly the [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) skill applied to a diff.

## Repo-specific review checklist

- [ ] New list/detail view goes through `restrict_queryset` (or a base class that does) and has a `no_permission` → 404 test.
- [ ] Writes touching related objects check permission on the *related* object (the change-type asymmetry, [03/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)).
- [ ] Events fired match intent; test asserts all four fields (actor/action_object/target/verb), not just count.
- [ ] Task kwargs changes are additive (in-flight message compatibility, [02/03](../02-stack-and-language-mastery/03-type-system-and-contracts.md)).
- [ ] New side effects: idempotent or reapable; cleanup on every exit path.
- [ ] Uses the right test base class (transaction vs plain).
- [ ] No `Document.objects` where `valid` is meant; correct manager choice.
- [ ] Migration is additive/reversible; renames are two-release.
- [ ] Matches house style; imports alphabetized; strings single-quoted.

## Example review comments (kind, specific, anchored)

> **Blocking.** This new `/reports/` view queries `Document.objects.all()`, so trashed documents leak into reports. Everywhere else uses `Document.valid` ([document_api_views.py L38](../../../mayan/apps/documents/api_views/document_api_views.py#L38)). Could we switch it and add a trashed-doc test?

> **Important.** The COUNT here runs per row on the list page (100 docs → 100 queries). The pattern for this is annotating in `get_source_queryset`. Want to pair on it?

> **Optional / nit.** House style favors keyword arguments even for single-arg calls — `get(pk=x)` rather than `get(x)`. Not blocking.

> **Question, not a request.** I might be missing context — why does this catch `Exception` rather than the specific `LockError`? The nearby tasks retry only named exceptions ([sources/tasks.py L88–L93](../../../mayan/apps/sources/tasks.py#L88-L93)); catching broadly could retry logic bugs.

Notice the shape: severity label, quantified impact, in-repo precedent, and an offer or a question. Lead with curiosity; reserve "blocking" for correctness and contract breaks.

## Receiving review (the other half)

Assume the reviewer is right until you've understood their point; if you disagree, disagree with evidence ("here's the test showing the current behavior"), not authority. Thank people for catching things. A PR is a conversation, not a defense.

**Drill:** take review kata 4 ([04/04](../04-code-reading-gym/04-review-katas.md) — the `is_stub` "fix") and write all three: the blocking comment, the question that surfaces the load-bearing behavior, and the concrete next step you'd propose. *Strong:* your comment makes the author *want* to write the deciding test, rather than feeling blocked.

**Interview angle:** code-review rounds ask you to review a diff live. The five layers + "lead with a question, quantify, cite precedent" is your framework. Timed katas: [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).
