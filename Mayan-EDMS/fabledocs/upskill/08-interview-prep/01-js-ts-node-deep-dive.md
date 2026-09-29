# JS / TS / Node deep-dive — question cards

14 cards. Mayan is Python, but the runtime *concepts* map directly; each card gives the JS framing and the repo anchor that makes your example concrete. Answer in the arc: definition → example → tradeoff → failure mode.

---

## Q1: Explain the Node event loop. How does a Python/Django app achieve concurrency differently?
**Round:** language/runtime. **Testing:** do you understand concurrency models, not just memorize "single-threaded."
**Repo anchor:** [task_manager/workers.py L12–L36](../../../mayan/apps/task_manager/workers.py#L12-L36) (process-based concurrency, per-worker `nice`/memory caps).
**Junior:** "Node is single-threaded with an event loop; Python uses threads."
**Mid adds:** Node interleaves via the loop + microtask queue; Django here is *synchronous per request* and gets concurrency from **multiple processes** (gunicorn workers + Celery workers), not `async`. "Don't block the loop" becomes "don't block a scarce worker" — same instinct, different unit.
**Senior adds:** the GIL only serializes threads within one process, irrelevant across worker processes; the real scaling knob is process/worker counts and queue topology; failure mode is worker-pool starvation by a long task, mitigated by tiering ([workers A–D](../01-codebase-cartography/04-runtime-and-tooling-map.md)).
**Follow-ups:** microtasks vs macrotasks; where would you put a CPU-bound task in Node? (worker_threads — the Node analog of Celery).
**Drill:** 90-second whiteboard of both models side by side.

## Q2: `Promise.all` vs `Promise.allSettled` vs a task queue with a completion step
**Testing:** parallel coordination + failure semantics.
**Repo anchor:** OCR chord [ocr/tasks.py L31–L48](../../../mayan/apps/ocr/tasks.py#L17-L49).
**Junior:** "`Promise.all` runs things in parallel."
**Mid adds:** `all` rejects on first failure (losing the rest); `allSettled` waits for all and reports each. Mayan's chord = `Promise.all` + a completion callback that runs once all header tasks succeed.
**Senior adds:** the chord's failure mode — one page exhausting retries means the finisher *never runs* ([Flow 5](../01-codebase-cartography/05-key-flows.md)), so partial completion needs its own visibility (per-page error rows); the `allSettled` analog is collecting per-unit results instead of failing the batch.
**Follow-up:** how do you cap concurrency in `Promise.all`? (a pool; Celery does it via worker concurrency).

## Q3: What is a closure? Show one that isn't a toy.
**Repo anchor:** handler factories [dynamic_search/handlers.py L56–L81](../../../mayan/apps/dynamic_search/handlers.py#L56-L81) (a function returning a handler that closes over `reverse_field_path`).
**Junior:** "a function that remembers its scope."
**Mid adds:** the factory pattern — parameterize behavior by returning a closure; the Python twin of the `var`-in-loop capture bug is solved exactly this way.
**Senior adds:** closures over mutable state = memory retention / stale-capture bugs; here each factory call binds a *fresh* value, which is the fix, not the bug.
**Drill:** write the loop-capture bug and the closure fix in JS, then point to the repo's factory as the same idea.

## Q4: `async/await` error handling vs Python exceptions — and why ordering matters
**Repo anchor:** the unreachable handler at [ocr/tasks.py L114–L131](../../../mayan/apps/ocr/tasks.py#L92-L131).
**Junior:** "wrap in try/catch."
**Mid adds:** `except`/`catch` clauses resolve in order; a broad handler before a specific one makes the specific one dead code — a real bug in this repo.
**Senior adds:** over-catching converts bugs to silence ([do_transition's blanket AttributeError](../../../mayan/apps/document_states/models/workflow_instance_models.py#L101-L104)); the discipline is catch-narrow, and in async retry contexts, retry only *transient* named errors.
**Follow-up:** unhandled promise rejection behavior in Node.

## Q5: Modules and side-effect imports
**Repo anchor:** registry-on-import ([events/classes.py L299–L303](../../../mayan/apps/events/classes.py#L287-L303)), URL wiring on `ready()` ([common/apps.py L83–L89](../../../mayan/apps/common/apps.py#L83-L89)).
**Junior:** "import brings in code."
**Mid adds:** imports *execute*; Mayan relies on this for wiring — creating an object at module scope registers it. ESM/CJS both execute, but most JS apps avoid load-bearing import side effects.
**Senior adds:** import order becomes semantics (`events` first in INSTALLED_APPS); circular-import avoidance via lazy `apps.get_model`/dotted-path strings; the tradeoff is magic-vs-greppability.
**Follow-up:** ESM vs CJS differences; tree-shaking implications of side-effectful modules.

## Q6: Decorators / higher-order functions
**Repo anchor:** [method_event, events/decorators.py L8–L33](../../../mayan/apps/events/decorators.py#L8-L33).
**Junior:** "a function that wraps another."
**Mid adds:** a decorator *factory* takes config and returns a decorator; `method_event` closes over the event config and commits an event after the wrapped method — the React HOC / Express-middleware pattern.
**Senior adds:** cross-cutting concerns (audit) centralized so call sites can't forget; cost is hidden behavior + the smuggled-actor tax ([_event_actor](../02-stack-and-language-mastery/02-framework-mental-models.md)).
**Drill:** implement `withTiming(fn)` in JS, then explain `method_event`'s two-phase pop.

## Q7: Idempotency and at-least-once delivery
**Repo anchor:** retry taxonomy [dynamic_search/tasks.py L60–L71](../../../mayan/apps/dynamic_search/tasks.py#L41-L89); non-idempotent upload [documents/tasks.py L141–L182](../../../mayan/apps/documents/tasks.py#L140-L182).
**Junior:** "idempotent means same result if you do it twice."
**Mid adds:** queues deliver at-least-once, so tasks must be idempotent or reapable; Mayan's search indexing overwrites (idempotent), but document upload has no dedupe key (a retry can duplicate).
**Senior adds:** the honest architecture is "at-least-once + idempotent-or-reaped," with periodic reapers as the safety net ([pattern 13](../03-architecture-and-patterns/05-pattern-catalog.md)); exactly-once is a myth you approximate with idempotency keys.
**Follow-up:** design an idempotency key for the upload endpoint.

## Q8: `unknown` vs `any`; typing a dynamic boundary
**Repo anchor:** the `ignore_results` typo [documents/tasks.py L140](../../../mayan/apps/documents/tasks.py#L140-L145) — the exact bug TS's strict object literals prevent.
**Junior:** "`any` turns off type checking."
**Mid adds:** `unknown` forces you to narrow before use; `any` opts out and propagates. Mayan has neither — so config typos fail silently at runtime; the mitigation is tests + reviewer knowledge.
**Senior adds:** where you'd add types first in an untyped codebase — the highest cross-boundary risk (task kwargs, serializer create/update, [02/03](../02-stack-and-language-mastery/03-type-system-and-contracts.md)).
**Follow-up:** how do you narrow `unknown` safely? (type guards).

## Q9: Generics and why they beat `any`
**Testing:** reusable type-safe abstractions. **Anchor:** conceptual — Mayan's registry `get(id)` returns loosely-typed objects; in TS you'd make `Registry<T>` return `T`.
**Junior:** "generics are like templates."
**Mid adds:** they preserve the relationship between input and output types (`identity<T>(x: T): T`); a registry typed `Map<string, EventType>` gives you `EventType` back, not `any`.
**Senior adds:** generic constraints (`<T extends Model>`), and when generics hurt readability (over-parameterization).
**Drill:** type Mayan's `EventType.get(id)` as a generic registry in TS.

## Q10: Narrowing, discriminated unions, exhaustiveness
**Anchor:** conceptual — Mayan's backend selection by dotted-path string is a stringly-typed union; TS would model backends as a discriminated union.
**Mid adds:** a `type Backend = FileLock | RedisLock | ModelLock` with a `kind` field lets the compiler force you to handle each; Mayan does it with runtime `get_backend()` and no exhaustiveness guarantee.
**Senior adds:** the `never` exhaustiveness trick in a `switch` default; the failure mode of stringly-typed dispatch (a new backend silently unhandled).

## Q11: `this` binding / method context
**Anchor:** Celery `bind=True` tasks receive `self` ([documents/tasks.py L19–L25](../../../mayan/apps/documents/tasks.py#L19-L25)) — a deliberate binding so `self.retry()` works.
**Junior:** "`this` depends on how you call it."
**Mid adds:** arrow functions capture lexical `this`; methods lose it when detached; Python is explicit (`self` is a parameter) which sidesteps the whole class of bugs — Celery opting into `self` shows why explicit binding is sometimes wanted.
**Senior adds:** `bind`/`call`/`apply`; why class methods passed as callbacks need binding.

## Q12: Event loop starvation / long tasks
**Anchor:** worker tiers keep OCR off interactive lanes ([workers.py](../../../mayan/apps/task_manager/workers.py#L12-L36)).
**Mid adds:** a CPU-bound task blocks the Node loop / a worker; solution is offload (worker_threads / a queue) + tiering by latency class.
**Senior adds:** backpressure (bounded queues), fairness, and per-child recycling for leaks — all present in Mayan's worker config.

## Q13: Memory leaks — how they happen and how you'd catch one
**Anchor:** `maximum_memory_per_child` recycles leaky workers ([workers.py L12–L36](../../../mayan/apps/task_manager/workers.py#L12-L36)); the test harness has descriptor/tempfile leak detectors ([testing/tests/mixins.py L250, L427](../../../mayan/apps/testing/tests/mixins.py)).
**Mid adds:** common causes (unbounded caches, retained closures, un-closed handles); Mayan mitigates in prod by recycling and in tests by *asserting* no leaks per test.
**Senior adds:** recycling is a band-aid — find the leak (heap snapshots / the file-descriptor assertions); leak detection *in the test suite* is unusually mature, worth citing.

## Q14: Streams / backpressure / bounded memory
**Anchor:** checksum streams the file in blocks ([document_file_models.py L199–L213](../../../mayan/apps/documents/models/document_file_models.py#L186-L213)); the `block_size=0` sentinel reads it all (memory spike).
**Mid adds:** streaming processes data in chunks so memory stays bounded — Mayan hashes large files block-by-block; the sentinel that disables the limit is the anti-example.
**Senior adds:** backpressure in streams (pause/resume); the tradeoff of the block-size setting (throughput vs memory); the `iterator()` in event export ([events/classes.py L62](../../../mayan/apps/events/classes.py#L41-L68)) as the DB-streaming analog.

**Self-grade for the set:** *Basic* — you give the definition for 10+. *Solid* — you give the arc (example+tradeoff+failure) for 10+. *Strong* — for 5 cards you can go two follow-ups deep without hedging. Record yourself on Q1, Q7, Q4 — those come up most.
