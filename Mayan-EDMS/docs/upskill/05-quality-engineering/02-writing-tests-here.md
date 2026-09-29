# Writing Tests Here

## Recipes

### Happy path upload

- Start from [`mayan/apps/sources/tests/test_web_form_source_api.py`](../../mayan/apps/sources/tests/test_web_form_source_api.py)
- Assert `202` on enqueue and resulting document/file existence.

### Validation failure

- Use checkout expiration rules from [`checkouts/models.py`](../../mayan/apps/checkouts/models.py#L68-L72)

### Permission failure

- Remove `permission_document_create` or source permission and assert denial.

### Cross-object permission failure

- Use cabinet tests where cabinet access and document access are distinct:
  [`mayan/apps/cabinets/tests/test_api.py`](../../mayan/apps/cabinets/tests/test_api.py)

### Async side effect

- Verify task submission method calls or resulting state in metadata/parsing tests.

### Migration behavior

- Mirror existing style in [`mayan/apps/documents/tests/test_migrations.py`](../../mayan/apps/documents/tests/test_migrations.py)

## Commands

```bash
# Inferred from Makefile.
python manage.py test mayan.apps.documents.tests.test_document_api \
  --settings=mayan.settings.testing.development --skip-migrations
python manage.py test mayan.apps.sources.tests.test_web_form_source_api \
  --settings=mayan.settings.testing.development --skip-migrations
python manage.py test --mayan-apps --settings=mayan.settings.testing.development --no-exclude --tag=migration_test
```
