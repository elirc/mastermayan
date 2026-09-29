# Maintainer communication

Mayan's own guidance sets the tone: issues are for **code problems only**, not support or deployment help; search existing issues first; be patient with a small core team; provide **repeatable** reproduction steps; never upload sensitive files — create a dummy file that triggers the same issue ([CONTRIBUTING.md](../../../CONTRIBUTING.md)). Every template below honors that.

## Asking for help without outsourcing your thinking

The rule: show your work, then ask the *narrow* question. Bad: "upload is broken, help." Good:

> Uploading via `POST /api/v4/documents/upload/` leaves the document as a stub. I traced to `task_document_file_upload` ([documents/tasks.py L52–L124](../../../mayan/apps/documents/tasks.py#L52-L124)); worker_c logs show `OperationalError` retried 3× then the task disappears — no `DocumentFile` created. Is the expected behavior that the staging file is deleted on final failure ([L110–L116](../../../mayan/apps/documents/tasks.py#L103-L116)), and if so how is the user meant to learn the upload failed? Repro: [steps].

That paragraph proves you narrowed it, cites lines, and asks one answerable question. Maintainers answer these fast because you've done the expensive part.

## Bug report template

```
**Summary:** one line.
**Environment:** version (4.3.1), DB, deploy (docker/local), lock backend.
**Steps to reproduce (repeatable):**
1. ...
**Expected:** ...
**Actual:** ... (status code / log excerpt / screenshot)
**Trace so far:** the file:line where you think it originates.
**Dummy file:** attached (never the real sensitive file).
```

## Feature proposal template

```
**Problem / use case:** who needs this and why (not "it'd be nice").
**Proposed behavior:** the smallest version that helps.
**Fit:** which existing pattern it follows (cite one, e.g. engine-backend
pattern for a new scanner).
**Alternatives:** what you considered.
**Willing to implement:** yes/with guidance/no.
```

Anchoring your proposal to an *existing pattern* ([pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md)) signals you'll build with the grain, not against it — the single biggest predictor of whether a maintainer engages.

## Respectful disagreement

When a reviewer is wrong (it happens), disagree with evidence and an out:

> I may be misreading — my understanding is that `Document.valid` already excludes trashed docs ([managers.py](../../../mayan/apps/documents/managers.py)), so the extra `in_trash=False` filter is redundant here. Test `test_...` passes without it. Happy to keep it if there's a case I'm missing?

Evidence (a passing test, an anchor), a genuine question, and a graceful concession path. Never "you're wrong"; always "here's what the code shows — what am I missing?"

## Responding to review on your PR

- Address every comment (fix, or reply with reasoning) — silence reads as ignoring.
- If asked to change something you disagree with, either do it or make the evidence-based case once; then defer. The maintainer owns the codebase's long-term shape.
- Thank people specifically ("good catch on the manager choice").
- Keep the PR focused; if review surfaces a bigger issue, file a follow-up rather than scope-creeping.

## Cultural specifics that will trip you up

- **CAA + DCO before code lands** — sort the paperwork early, not after your PR is approved.
- **`development`, not `master`** — targeting the wrong branch is an instant bounce.
- **Support questions go to the forum**, not issues — posting "how do I configure X" as an issue burns goodwill.
- **Patience** — small team; a slow response is not a no.

**Drill:** write the bug report for the version-modification silent failure ([05/03 scenario 2](../05-quality-engineering/03-systematic-debugging.md)), including a dummy-file description and the one narrow question. Then write the respectful-disagreement reply for a hypothetical reviewer who says "silent failure is fine, it's an admin action." *Strong:* your disagreement reframes it as a *user-trust* problem with a concrete cost, and still offers to scope the notification separately if they prefer.

**Interview angle:** behavioral rounds ask about conflict and asking for help. "Tell me about a disagreement with a senior engineer" → the evidence-based-disagreement pattern. STAR worksheets: [08/06](../08-interview-prep/06-behavioral-star-stories.md).
