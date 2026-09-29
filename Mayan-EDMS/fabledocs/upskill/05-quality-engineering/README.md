# 05 — Quality engineering

How this repo keeps 2,181 Python files honest — and how you practice the same skills.

| File | Focus |
| --- | --- |
| [01-testing-strategy.md](01-testing-strategy.md) | The layer model, the harness machinery, what not to test |
| [02-writing-tests-here.md](02-writing-tests-here.md) | Seven concrete recipes with the house helpers |
| [03-systematic-debugging.md](03-systematic-debugging.md) | The method + five repo-specific scenarios |
| [04-performance-thinking.md](04-performance-thinking.md) | Where this app actually gets slow, and how to prove it |
| [05-security-checklist.md](05-security-checklist.md) | Mapped to real code; ends with a pre-merge checklist |
| [06-observability-and-operations.md](06-observability-and-operations.md) | "How would I know this broke?" per flow |

The unifying idea, and the interview line worth memorizing: **quality is a set of feedback loops, ranked by latency** — type/lint (seconds) is absent here, tests (minutes) are strong, review (hours) is conventions, production error logs (days) are per-entity. When one loop is missing, the others must tighten. This repo compensates for no-static-typing with an unusually disciplined test harness; know that tradeoff story.
