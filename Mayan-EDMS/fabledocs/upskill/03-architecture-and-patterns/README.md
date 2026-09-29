# 03 — Architecture and patterns

The judgment module. Cartography told you *where*; this tells you *why*, *what it costs*, and *when it fails*.

| File | Focus |
| --- | --- |
| [01-boundaries-and-layers.md](01-boundaries-and-layers.md) | Who owns what; two clean boundaries and three leaks |
| [02-data-model-and-persistence.md](02-data-model-and-persistence.md) | Entities, transactions, consistency, safe schema change |
| [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) | The three-tier authorization machine and its edges |
| [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) | Every side effect mapped; idempotency, retries, locks |
| [05-pattern-catalog.md](05-pattern-catalog.md) | 16 pattern cards — recognition training |
| [06-architecture-critique.md](06-architecture-critique.md) | Strengths, risks, and what I'd change owning this for 3 months (doubles as system-design interview prep) |

Senior vocabulary is used throughout and defined in place: **invariant** (must always hold), **boundary** (where responsibility changes hands), **contract** (what a boundary promises), **idempotency** (safe to repeat), **blast radius** (how far a failure spreads). These five words, used precisely with repo examples, are worth more in a mid-level interview than any framework trivia.
