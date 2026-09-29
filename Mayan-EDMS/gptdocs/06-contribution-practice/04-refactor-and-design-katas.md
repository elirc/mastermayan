# Refactor and design katas

1. Boundary leak: move cleanup policy out of an upload task incrementally without changing events/retries.
2. Outbox: design DB→broker delivery with at-least-once publishing and consumer dedupe.
3. Split module: extract checksum/integrity behavior while preserving storage hooks.
4. Type safety: model source action arguments as explicit runtime schemas and generated TS unions.
5. Migration: add operation status expand/backfill/contract with old workers still draining.
6. N+1: benchmark an ACL-heavy list, propose query change, prove permission equivalence.
7. RFC: compare direct upload through web versus signed object-storage upload.
8. Review: reject a “cleanup” PR that changes 404 authorization behavior accidentally.

Self-grade: **Basic** produces a plausible end state. **Solid** preserves contracts with tests and staged migration. **Strong** defines invariant, competing options, operational failure, security, measurement, rollback, and why maintainers should accept the incremental path.

Interview story potential: each kata can become a tradeoff story only after you implement or simulate evidence; never claim repository changes you did not make.
