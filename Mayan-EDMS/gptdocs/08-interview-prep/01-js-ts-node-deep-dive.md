# JavaScript/TypeScript/Node deep dive through a Python system

## Q1: How would blocking work differ between Node and this Django/Celery architecture?
Round: JS/TS deep-dive. Testing: event loop versus process/task concurrency. Repo anchor: [OCR chord](../../mayan/apps/ocr/tasks.py#L17-L48). Junior: “Node is async.” Mid: CPU work blocks Node; offload OCR and bound concurrency. Senior: adds worker isolation, backpressure, cancellation, and cost. Follow-up: when are worker threads enough? Drill: 90 seconds.

## Q2: Why pass IDs to a background job instead of objects or file bytes?
Round: JS/TS deep-dive. Repo anchor: [upload task](../../mayan/apps/documents/tasks.py#L52-L72). Junior: serialization. Mid: small stable payload, reload current state; IDs can go stale. Senior: payload versioning, idempotency, tenancy, retention. Follow-up: what if row disappears? Drill: define policy.

## Q3: Which errors should an async job retry?
Round: JS/TS deep-dive. Repo anchor: [OperationalError retry](../../mayan/apps/documents/tasks.py#L64-L80). Junior: temporary errors. Mid: retry classified transient errors with bounds/backoff. Senior: jitter, retry budget, poison handling, side-effect safety. Follow-up: HTTP 429 versus 400. Drill: classify ten errors.

## Q4: Explain fan-out/fan-in and its failure modes.
Round: JS/TS deep-dive. Repo anchor: [OCR page chord](../../mayan/apps/ocr/tasks.py#L30-L43). Junior: pages run in parallel. Mid: callback waits; slow/failing page gates completion. Senior: partial results, result backend, fairness, cancellation. Follow-up: implement with `Promise.allSettled`. Drill: whiteboard.

## Q5: What is idempotency, and where would you need it here?
Round: JS/TS deep-dive. Repo anchor: [file upload worker](../../mayan/apps/documents/tasks.py#L48-L124). Junior: duplicate calls do not duplicate. Mid: broker redelivery may create two versions. Senior: operation key, unique constraint, state machine, replay semantics. Follow-up: idempotent versus exactly once. Drill: propose key.

## Q6: What closure bug could appear when creating per-page jobs in JavaScript?
Round: JS/TS deep-dive. Repo anchor: conceptual parallel to [page loop](../../mayan/apps/ocr/tasks.py#L31-L38). Junior: `var` captures final index. Mid: use `let`/value binding and test completion mapping. Senior: warns ordering and mutable shared state. Follow-up: microtask timing. Drill: write fake JS.

## Q7: When should independent async calls be serial versus parallel?
Round: JS/TS deep-dive. Repo anchor: [OCR independent pages](../../mayan/apps/ocr/tasks.py#L31-L43). Junior: parallel is faster. Mid: only independent work; bound concurrency. Senior: dependencies, rate limits, memory, fairness, tail latency. Follow-up: upload then enqueue? Drill: label dependencies.

## Q8: Why does a TypeScript type not validate an API request?
Round: JS/TS deep-dive. Repo anchor: [runtime serializer](../../mayan/apps/documents/serializers/document_file_serializers.py#L71-L140). Junior: types disappear. Mid: parse `unknown` at boundary. Senior: schema ownership, versioning, generated types/drift. Follow-up: where validate twice? Drill: define Zod-style schema.

## Q9: `unknown` versus `any` for source action arguments?
Round: JS/TS deep-dive. Repo anchor: [JSON arguments](../../mayan/apps/sources/serializers.py#L49-L63). Junior: `unknown` requires checking. Mid: discriminate by action and validate. Senior: plugin schema registry and compatibility. Follow-up: safe narrowing. Drill: make union.

## Q10: How would discriminated unions model backend actions?
Round: JS/TS deep-dive. Repo anchor: [action discovery](../../mayan/apps/sources/serializers.py#L27-L46). Junior: `type` field selects shape. Mid: exhaustive switch and runtime schema. Senior: open plugin ecosystem makes closed unions/versioning hard. Follow-up: `never` exhaustiveness. Drill: two actions.

## Q11: Explain module initialization and circular dependencies.
Round: JS/TS deep-dive. Repo anchor: Django delays models with [app registry lookup](../../mayan/apps/documents/tasks.py#L56-L72). Junior: modules load imports. Mid: top-level execution/order and partially initialized cycles. Senior: inversion/registries have runtime-discovery costs. Follow-up: CommonJS versus ESM. Drill: draw graph.

## Q12: How should an API propagate errors from background work?
Round: JS/TS deep-dive. Repo anchor: request returns [202 before work](../../mayan/apps/documents/api_views/document_file_api_views.py#L38-L60). Junior: show error. Mid: operation resource with terminal failure. Senior: stable error taxonomy, privacy, retries, support correlation, retention. Follow-up: webhook/SSE/polling. Drill: sketch response.
