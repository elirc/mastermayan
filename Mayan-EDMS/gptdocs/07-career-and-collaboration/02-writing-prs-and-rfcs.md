# Writing PRs and RFCs

PR template:

- What/why: user or maintainer outcome.
- Approach: boundary and existing pattern followed.
- Tests: exact commands and negative/failure cases.
- Risks: contract, permission, data, async, performance, rollout.
- Rollback: code/data/queue compatibility.
- Not included: explicit scope boundary.

Use an RFC when choices affect public contracts, schema/data migration, cross-app ownership, operations, security model, or weeks of work. RFC: context → requirements/non-goals → current evidence → options → decision → API/data model → failure modes → security/performance/observability → migration/rollout/rollback → test plan → open questions.

Commit messages describe the invariant, e.g. `test(documents): reject file IDs outside URL parent`.
