# Language and runtime model (for the Node engineer)

## 1. Concurrency: processes, not an event loop

Node gives you one thread + an event loop; concurrency is `await`. This codebase has **zero** `async def` — concurrency comes from *processes*: multiple gunicorn web workers, multiple Celery worker processes ([task_manager/workers.py](../../../mayan/apps/task_manager/workers.py#L12-L36) sets per-worker concurrency and even OS `nice` levels). Inside one request or one task, code is straight-line blocking: `checksum_update` reads a file in a plain `while True` loop ([document_file_models.py](../../../mayan/apps/documents/models/document_file_models.py#L199-L213)) and nothing else runs on that worker meanwhile.

Consequences you must internalize:
- "Don't block the event loop" becomes **"don't block a scarce worker"** — same instinct, different unit. A slow OCR in the web process would freeze one of ~N workers; that's why it's a queued task (Flow 5).
- Parallelism across tasks is real (separate processes; the GIL only serializes threads *within* one process, and this repo barely uses threads — one exception: the module-level `threading.Lock` guarding the file-lock backend, [file_lock.py](../../../mayan/apps/lock_manager/backends/file_lock.py#L19)).
- Shared mutable state cannot live in process memory. Where Node devs reach for a module-level cache, this repo uses Redis/DB. When it *does* use module-level state — the class registries ([events/classes.py](../../../mayan/apps/events/classes.py#L287-L303)) — that state is **write-once at import time**, which is the only safe kind.

Failure mode to name in interviews: the Celery **prefetch + long task** starvation problem, and per-child memory caps (`maximum_memory_per_child`, [workers.py](../../../mayan/apps/task_manager/workers.py#L12-L18)) as the leak mitigation — the Python analog of restarting a leaky Node pod, built into the worker.

## 2. Imports are execution (and this repo weaponizes it)

`require()`/ESM imports also execute modules, but Node apps rarely rely on import side effects. Here, importing an app's modules **is** the wiring: creating a `SearchModel` at module scope registers it ([documents/search.py](../../../mayan/apps/documents/search.py#L22-L28)); `EventTypeNamespace.__init__` writes into a class-level `_registry` dict ([events/classes.py](../../../mayan/apps/events/classes.py#L299-L303)); every app's `ready()` hook mutates the global urlconf ([common/apps.py](../../../mayan/apps/common/apps.py#L83-L89)).

Sharp edges:
- **Import order matters.** `events` is deliberately first in `INSTALLED_APPS` "so it can preload all events" ([settings/base.py](../../../mayan/settings/base.py#L42-L48)).
- **Circular imports** are dodged three ways you'll see constantly: `apps.get_model('documents', 'DocumentFile')` at call time ([document_models.py](../../../mayan/apps/documents/models/document_models.py#L201-L203)), "hidden imports" inside functions ([events/classes.py](../../../mayan/apps/events/classes.py#L227-L229)), and dotted-path strings resolved with `import_string` ([documents/tasks.py](../../../mayan/apps/documents/tasks.py#L187-L193)).
- Test pollution: registries persist across tests within a process — the test harness has machinery to cope (see [05-quality/01-testing-strategy.md](../05-quality-engineering/01-testing-strategy.md)).

JS analogy that holds: NestJS decorators + DI container do the same "declare at import, resolve at runtime" dance. Analogy that breaks: there's no container — the registry *is* a global dict, and nothing namespaces it per test or per tenant.

## 3. Decorators = higher-order functions with syntax

You already know this pattern as `withAuth(handler)`. Read [`method_event`](../../../mayan/apps/events/decorators.py#L8-L33) closely — it's a decorator **factory** (takes config, returns decorator) closing over `event_manager_class` and kwargs, wrapping methods so an event commits after the wrapped call. Same shape as Celery's `@app.task(bind=True, ...)` ([documents/tasks.py](../../../mayan/apps/documents/tasks.py#L19-L25)) and the lock decorators (`@locked_class_method`, [file_caching/models.py](../../../mayan/apps/file_caching/models.py#L374-L387)).

Closure gotcha, Python edition: late binding in loops (the Python twin of the `var` loop bug). This repo mostly sidesteps it with factory functions — `handler_factory_index_related_instance_save(reverse_field_path)` returns a fresh handler closing over its argument ([dynamic_search/handlers.py](../../../mayan/apps/dynamic_search/handlers.py#L56-L81)). That is *exactly* your "IIFE to capture the loop variable" pattern, promoted to architecture.

## 4. Exceptions are the error channel — and ordering is a contract

No `Result` types, no error-first callbacks: control flow uses exceptions, including for expected cases (`LockError` means "someone else is working," [sources/tasks.py](../../../mayan/apps/sources/tasks.py#L25-L33)). Two rules with in-repo evidence:

- **except clauses are checked in order; put subclasses first.** The OCR finisher lists `except Exception` before `except OperationalError`, making the second handler unreachable ([ocr/tasks.py](../../../mayan/apps/ocr/tasks.py#L114-L131)) — a real, shippable find.
- **Catching too much converts bugs into silence.** `do_transition`'s blanket `except AttributeError` ([workflow_instance_models.py](../../../mayan/apps/document_states/models/workflow_instance_models.py#L101-L104)) intends "workflow lacks initial state" but also swallows attribute typos in any state action. The `try/finally` discipline around locks ([sources/tasks.py](../../../mayan/apps/sources/tasks.py#L35-L55)) is the positive example.

## 5. Context managers = `try/finally` as a reusable object

`with shared_uploaded_file.open() as f:` guarantees release the way your `finally { release() }` does, but composably. This repo also *builds* them: `@contextmanager def create_file(...)` yields a writable cache file inside a lock ([file_caching/models.py](../../../mayan/apps/file_caching/models.py#L225-L272)) — acquire → prune → yield → close/update-size → release, with the error path cleaning up partial state. When you see `yield` inside `@contextmanager`, read it as "the caller's block runs here."

## Pitfall checklist (tape to monitor)

- [ ] Mutable default arguments (`def f(x, cache={})`) — the repo avoids them; you should too.
- [ ] Class attributes are shared: `_hooks_pre_create = []` on `Document` ([document_models.py](../../../mayan/apps/documents/models/document_models.py#L51)) is *intentionally* shared across instances — that's the registry idiom, not a bug; copying that shape for per-instance state would be one.
- [ ] Truthiness: empty querysets are falsy, and evaluating one **runs SQL** ([acls/managers.py](../../../mayan/apps/acls/managers.py#L119-L123)).
- [ ] `str` vs bytes at file boundaries (`force_text`/`force_bytes` calls sprinkled through storage code).
- [ ] Exception ordering (see above).

## Interview angle

1. *"Node handles 10k concurrent connections on one thread — how does a Python/Django app cope?"* → processes + queues; know the worker table.
2. *"Explain a decorator you've read, not written."* → `method_event`, factory + closure + `functools.wraps`.
3. *"What's the GIL and when does it matter?"* → threads within a process; irrelevant across Celery worker processes; mention the one real thread lock in `file_lock.py`.
4. *"Tell me about a bug caused by exception handling."* → unreachable `except OperationalError` in OCR finisher; broad `AttributeError` swallow in workflows.

Cards: [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q1–Q6.

**Drill:** rewrite [`task_source_process_document`](../../../mayan/apps/sources/tasks.py#L18-L55)'s lock handling as JS pseudo-code with `try/finally`, then list what the Python version's `else:` clause buys (lock-acquisition failure exits *without* touching the source — no accidental work outside the lock). *Strong:* you also spot that if `Source.objects.get` raises inside the guarded block, the `except` handler references an unbound `source` variable.
