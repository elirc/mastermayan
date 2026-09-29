# Risk Register

| Risk | Evidence | File anchors | Impact | Likelihood | Suggested test | Suggested fix | Confidence |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Heavy synchronous upload-side derived work | `DocumentFile.save()` computes many fields and pages inline | [`document_file_models.py`](../../mayan/apps/documents/models/document_file_models.py#L460-L484) | request latency, brittle failures | Medium | upload integration timing/failure tests | consider phased async extraction | High |
| Upload temp-file orphan possibility | cleanup after downstream call | [`sources/tasks.py`](../../mayan/apps/sources/tasks.py#L95-L103) | storage leak | Medium | failure-injection test | sweeper or stronger cleanup path | Medium |
| Signal/task amplification in indexing | document events enqueue tasks | [`document_indexing/handlers.py`](../../mayan/apps/document_indexing/handlers.py#L71-L83) | queue spikes | Medium | bulk-document operation test | batch-aware coalescing | Medium |
| Permission regression via raw PK fetch | repeated ACL-filter pattern is easy to bypass accidentally | [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L79-L86) | security issue | High | permission regression tests | helper abstraction + review checklist | High |
| AJAX race/stale content | front-end request cancellation/throttling complexity | [`partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L96-L151) | confusing UI state | Medium | browser/manual interaction test | stronger navigation state handling | Medium |
