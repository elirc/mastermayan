# 01 — Codebase cartography

You cannot form judgment about code you cannot find. This module teaches you to navigate 57 Django apps and 2,181 Python files the way a senior does: by learning the *shape* once, then predicting where anything lives.

| File | What you get |
| --- | --- |
| [01-system-map.md](01-system-map.md) | The repo's shape, ownership map, and public-vs-private surfaces |
| [02-file-reading-order.md](02-file-reading-order.md) | 30 files in reading order, with junior/mid/senior paths |
| [03-domain-glossary.md](03-domain-glossary.md) | Document vs DocumentFile vs DocumentVersion and other traps |
| [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) | Processes, queues, settings, and the build/test toolchain |
| [05-key-flows.md](05-key-flows.md) | Eight end-to-end traces — the backbone of the whole curriculum |

Transferable skill: every mature codebase has a *repeated per-module skeleton*. Here it is `models/ views/ api_views/ serializers/ tasks/ events.py permissions.py queues.py urls.py tests/` inside each app ([documents/](../../../mayan/apps/documents/) is the canonical example). Once you've internalized one app's skeleton, you've internalized fifty-seven. In an interview, "how do you approach an unfamiliar codebase?" is answered exactly this way: find the skeleton, find the registration points, trace one flow end to end. See [Q-card practice](../08-interview-prep/README.md).
