# Good first tickets

Use the common plan: reproduce/characterize → make the smallest change → run targeted test → document risk. A maintainer may reject any ticket that changes a public contract, adds dependency, lacks regression proof, or expands scope.

| # | Ticket (difficulty/time) | Why useful / anchors / checks | Interview story potential |
| --- | --- | --- | --- |
| 1 | Wrong-parent download test (Easy, 2h) | Protects IDOR invariant; [queryset](../../mayan/apps/documents/api_views/document_file_api_views.py#L119-L120); API test | Security boundary |
| 2 | Wrong-parent detail test (Easy, 2h) | Low-risk test-only characterization | Defense in depth |
| 3 | Upload validation side-effect test (Easy, 2h) | Assert 400 creates no staging data; [create](../../mayan/apps/documents/api_views/document_file_api_views.py#L38-L47) | Contract testing |
| 4 | Enqueue-failure characterization (Medium, 4h) | Documents orphan behavior without assuming fix | Failure injection |
| 5 | Upload 202 contract test (Easy, 2h) | Preserves async semantics | API tradeoff |
| 6 | Download event assertion cleanup (Easy, 2h) | Follow [existing assertions](../../mayan/apps/documents/tests/test_document_file_api.py#L160-L186) | Auditability |
| 7 | Anonymous ACL list test (Easy, 3h) | Protects `none()` behavior [ACL](../../mayan/apps/acls/managers.py#L268-L270) | Authorization |
| 8 | Source lock contention test (Medium, 4h) | One backend call under contention; [task](../../mayan/apps/sources/tasks.py#L18-L55) | Concurrency |
| 9 | Source error-log test (Easy, 3h) | Visible failure contract | Operations thinking |
| 10 | OCR cache-miss retry test (Medium, 4h) | Protect transient retry [task](../../mayan/apps/ocr/tasks.py#L75-L89) | Async debugging |
| 11 | OCR completion event test (Easy, 3h) | Audit/event contract [finish](../../mayan/apps/ocr/tasks.py#L92-L125) | Eventing |
| 12 | Checksum empty-file test (Easy, 2h) | Stream-loop edge case [model](../../mayan/apps/documents/models/document_file_models.py#L186-L213) | Data integrity |
| 13 | Checksum block-size test (Medium, 3h) | Proves bounded reads | Performance proof |
| 14 | File delete storage-failure characterization (Medium, 4h) | Maps compensation gap | Reliability |
| 15 | Toolchain mismatch note (Easy, 1h) | Clarifies `tox.ini` versus `setup.py`; docs only | Repository orientation |
| 16 | Targeted test command doc (Easy, 1h) | Makes contribution cheaper | Developer experience |

Acceptance criteria for each: test fails for the intended reason before change, passes afterward, no unrelated formatting, exact command reported, and no production behavior invented. Read the anchor first; likely files are its adjacent test module and mixins.
