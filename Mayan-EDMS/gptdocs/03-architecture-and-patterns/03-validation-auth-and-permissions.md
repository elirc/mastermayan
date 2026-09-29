# Validation, authentication, and authorization

Authentication answers who; authorization answers whether that identity may act on this resource. Serializer validation accepts the upload shape at [document_file_serializers.py](../../mayan/apps/documents/serializers/document_file_serializers.py#L71-L140). Views declare method-specific permissions; `MayanPermission` separates view/global checks from object ACL checks at [rest_api/permissions.py](../../mayan/apps/rest_api/permissions.py#L9-L59). ACL filtering denies anonymous users and restricts querysets at [acls/managers.py](../../mayan/apps/acls/managers.py#L268-L294).

What juniors miss: a valid ID is not authorization, list endpoints leak too, parent/child IDs must agree, and 404 can intentionally hide existence. What seniors check: every access path, default behavior when a declaration is absent, inheritance, global role bypass, cache scope, audit, bulk endpoints, background tasks, and cross-parent tests.

Pre-merge: validate shape/size/content; scope parent; require action permission; avoid revealing existence; test no permission, permission, wrong parent, anonymous, and privileged role.
