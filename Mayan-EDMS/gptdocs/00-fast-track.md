# Weekend fast track

## Day 1: establish the map

1. Read [README.rst](../README.rst#L9-L24), [setup.py](../setup.py#L62-L110), [base.py](../mayan/settings/base.py#L269-L300), and [celery.py](../mayan/celery.py#L1-L11).
2. Install with `make setup-dev-environment` (**inferred**, not run). On Windows, prefer the repository's Docker path; native dependencies may not be portable.
3. Run `make runserver` (**inferred**) and `make test MODULE=documents.tests.test_document_file_api` (**inferred**).
4. Trace file upload: serializer → API view → staged file → task → domain method. Then trace ACL denial: permission declaration → DRF permission → ACL manager → 404 test.

The first ten files, in order:

1. `README.rst` — product boundaries.
2. `setup.py` — runtime and integrations.
3. `mayan/settings/base.py` — framework composition.
4. `mayan/celery.py` — worker bootstrap.
5. `mayan/apps/documents/api_views/document_file_api_views.py` — HTTP boundary.
6. `mayan/apps/documents/serializers/document_file_serializers.py` — API contract.
7. `mayan/apps/documents/tasks.py` — async hand-off.
8. `mayan/apps/documents/models/document_file_models.py` — persistence/file invariant.
9. `mayan/apps/rest_api/permissions.py` — security adapter.
10. `mayan/apps/documents/tests/test_document_file_api.py` — executable expectations.

## Day 2: make the system explainable

A safe first change is a test-only case proving unauthorized upload returns 404 and creates neither a new file nor an event. It follows the paired tests in [test_document_file_api.py](../mayan/apps/documents/tests/test_document_file_api.py#L254-L260) and has low blast radius because production code is untouched.

Teach-back: “Why does file upload return 202, where is authorization enforced, what can fail after the response, and how would the user know?” Speak for 90 seconds without saying “the framework handles it.”

Self-grade: **Basic** names the view and task. **Solid** explains the staged-file ownership transfer and object permission. **Strong** identifies retry/idempotency/observability risks and proposes a regression test.

This path skips exhaustive app discovery, schema migrations, template rendering, deployment, search internals, and most integrations.
