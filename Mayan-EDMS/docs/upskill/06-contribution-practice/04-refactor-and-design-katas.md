# Refactor and Design Katas

## Kata: Identify a boundary leak

- Candidate: `DocumentFile.save()` doing heavy derived work.
- Self-grade:
  Weak: "too much logic."
  Solid: names specific responsibilities.
  Strong: proposes phased extraction with tests and rollback.

## Kata: Propose an outbox-like improvement

- Candidate: signal-driven indexing and upload callbacks.

## Kata: Split a large module

- Candidate: [`mayan_app.js`](../../../mayan/apps/appearance/static/appearance/js/mayan_app.js)

## Kata: Improve type safety without static types

- Candidate: replace loosely shaped callback kwargs with a stricter helper object or validation layer.

## Kata: Reduce N+1 or task storm risk

- Candidate: indexing handlers on bulk document changes.

## Kata: Write an RFC

- Topic: upload observability or document-file save-path refactor.
