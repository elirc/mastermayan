# Testing strategy

Unit-test pure transformations and domain invariants; API-test serialization, status, permissions, scoping, and events; task-test retry/cleanup/idempotency; integration-test DB/storage/broker boundaries; migration-test old data. Test helpers form a small DSL at [document_file_mixins.py](../../mayan/apps/documents/tests/mixins/document_file_mixins.py#L16-L68). Paired permission tests demonstrate expected behavior at [test_document_file_api.py](../../mayan/apps/documents/tests/test_document_file_api.py#L27-L74).

Do not test Django internals or duplicate every serializer field at every layer. Control time/randomness, isolate storage, drain eager tasks intentionally, assert query counts where performance matters, and ensure cleanup. Flake warning signs: shared globals, unordered querysets, real clocks/services, and tasks escaping the test transaction.
