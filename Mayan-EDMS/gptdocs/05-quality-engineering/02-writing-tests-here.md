# Writing tests here

Recipes, all commands inferred:

- Happy API: use `BaseAPITestCase`, request mixin, grant ACL, assert response/data/events. `make test MODULE=documents.tests.test_document_file_api`.
- Validation failure: omit `file_new` or use invalid `action`; assert 400 and no staging/file/event.
- Permission failure: no grant; assert 404 and unchanged counts, following [the existing pair](../../mayan/apps/documents/tests/test_document_file_api.py#L27-L74).
- Cross-parent rejection: grant document A, request file B under A; assert 404.
- Enqueue failure: mock `apply_async`; assert the intended staging cleanup contract.
- Retry: raise `OperationalError`; assert retry and staged data retained.
- Duplicate delivery: invoke task twice with one operation ID; define one outcome.
- Download audit: assert bytes/headers and exactly one event, following [lines 160–186](../../mayan/apps/documents/tests/test_document_file_api.py#L160-L186).

Strong tests state the invariant in their name and assert both expected output and forbidden side effects.
