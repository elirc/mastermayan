# Language and Runtime Model

## Python data-model and object lifecycle

Concept: Python classes here are not just data bags; model methods carry business invariants and side effects.

Why it matters in production: If you treat Django models like passive records, you will bypass logic such as rollback, event emission, derived fields, and guard hooks.

Real code:

- Document creation orchestration in [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L138-L176)
- Document file save lifecycle in [`document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py#L428-L505)
- Checkout invariant in [`checkouts/models.py`](../../mayan/apps/checkouts/models.py#L68-L116)

Failure modes:

- Creating rows through shortcuts that bypass expected methods.
- Assuming `save()` is cheap.
- Forgetting that signal and hook chains amplify a single call.

Drill:

- Read `Document.file_new()` and list every side effect before and after the `DocumentFile` row exists.

Self-grade:

- Weak: "it saves a file."
- Solid: names checksum, mimetype, page count, signals, and label updates.
- Strong: also identifies transaction boundaries and where rollback is incomplete.

## Async Python and task boundaries

Concept: request code passes primitive identifiers to tasks; workers rehydrate state.

Why it matters in production: serializing live objects across process boundaries is brittle and stale; IDs plus reloads make retries possible.

Real code:

- [`sources/tasks.py`](../../mayan/apps/sources/tasks.py#L58-L103)
- [`file_metadata/tasks.py`](../../mayan/apps/file_metadata/tasks.py#L17-L48)
- [`document_parsing/tasks.py`](../../mayan/apps/document_parsing/tasks.py#L11-L34)
- [`ocr/tasks.py`](../../mayan/apps/ocr/tasks.py#L17-L131)

Pitfall checklist:

- Do not pass model instances to Celery.
- Do not assume task input is still valid when it runs.
- Recreate user context explicitly with `user_id` if you need auditability.
- Separate retryable infrastructure failures from permanent business failures.

## JavaScript runtime mental model in this repo

Concept: the JS surface is older jQuery-style event orchestration, not React. State lives in DOM, browser history, and singleton-like objects.

Real code:

- AJAX navigation and throttling in [`partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L87-L152)
- UI event wiring in [`mayan_app.js`](../../mayan/apps/appearance/static/appearance/js/mayan_app.js#L53-L69) and [`mayan_app.js`](../../mayan/apps/appearance/static/appearance/js/mayan_app.js#L367-L420)
- form affordances in [`metadata_form.js`](../../mayan/apps/metadata/static/metadata/js/metadata_form.js#L3-L8) and [`tags_form.js`](../../mayan/apps/tags/static/tags/js/tags_form.js#L3-L24)

Why it matters: interview prep often focuses on modern frameworks, but mid-level judgment includes understanding event delegation, AJAX race handling, and DOM-driven state in legacy but still valuable code.

Failure modes:

- stale DOM assumptions after partial replacement
- multiple overlapping AJAX responses
- event handlers bound to ephemeral nodes instead of delegated parents

Drill:

- Explain why `PartialNavigation.setupAjaxAnchors()` uses delegated click handling on `body` instead of attaching listeners to each link.

## Verification Notes

- Focused on repo-native runtime patterns instead of generic Python tutorials.
- JavaScript examples are limited because this repo’s client surface is modest compared with the server side.
