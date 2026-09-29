# Annotation drills

For each anchor, annotate inputs, outputs, dependencies, invariant, side effects, failure modes, and tests.

1. Upload view: [document_file_api_views.py](../../mayan/apps/documents/api_views/document_file_api_views.py#L28-L65). Where does request ownership end?
2. Upload serializer: [document_file_serializers.py](../../mayan/apps/documents/serializers/document_file_serializers.py#L71-L140). Which fields are public writes?
3. Upload worker: [documents/tasks.py](../../mayan/apps/documents/tasks.py#L48-L124). Build an exception/cleanup matrix.
4. Checksum loop: [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L186-L213). What bounds memory?
5. File deletion: [document_file_models.py](../../mayan/apps/documents/models/document_file_models.py#L215-L236). Which side effects cannot roll back together?
6. Permission adapter: [rest_api/permissions.py](../../mayan/apps/rest_api/permissions.py#L9-L59). What is the default if no map entry exists?
7. ACL restriction: [acls/managers.py](../../mayan/apps/acls/managers.py#L268-L294). When is the full queryset returned?
8. Source dispatch: [sources/api_views.py](../../mayan/apps/sources/api_views.py#L42-L62). Which input wins on key collision?
9. Source lock: [sources/tasks.py](../../mayan/apps/sources/tasks.py#L18-L55). What releases it after exceptions?
10. OCR chord: [ocr/tasks.py](../../mayan/apps/ocr/tasks.py#L17-L48). What does completion depend on?

Rubric: **Basic** labels inputs/outputs. **Solid** traces transformations and names validation/authorization. **Strong** states invariant, failure domain, regression test, and alternative design.
