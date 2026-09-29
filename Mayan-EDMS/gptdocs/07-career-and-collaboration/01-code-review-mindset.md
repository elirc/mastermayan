# Code review mindset

Review in layers: does it work → is it correct/secure → will it stay correct under retry/concurrency → does it fit repository patterns → is it kind to future maintainers? Repo checklist: permission map, parent scoping, serializer contract, query count, task retry class, idempotency, staging cleanup, event semantics, migrations, generated files, targeted tests, operations visibility.

Good comment: “This global child lookup bypasses the parent scope used at [document_file_api_views.py](../../mayan/apps/documents/api_views/document_file_api_views.py#L89-L120). Could we keep the scoped queryset and add a wrong-parent test? I consider this blocking because an authorized user may access a file through an unrelated document URL.”

Avoid labels without consequences (“bad abstraction”). State observation, user/system risk, evidence, requested outcome, and optional implementation idea.
