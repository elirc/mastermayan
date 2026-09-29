# Writing tests here — seven recipes

Run commands are `inferred` (Makefile-derived); test code shapes are copied from real house patterns. Target file: your app's `tests/test_*.py`, base classes from [documents/tests/base.py](../../../mayan/apps/documents/tests/base.py).

## Recipe 1: happy-path model behavior

```python
# Illustrative fake code: not from this repo (imitates test_document_models.py)
from .base import GenericDocumentTestCase

class DocumentTrashTestCase(GenericDocumentTestCase):
    def test_first_delete_trashes(self):
        self._test_document.delete()
        self.assertTrue(self._test_document.in_trash)
        self.assertEqual(Document.valid.count(), 0)
        self.assertEqual(Document.objects.count(), 1)
```

`DocumentTestMixin` auto-uploads a test document unless `auto_upload_test_document = False` ([test_document_api.py L25](../../../mayan/apps/documents/tests/test_document_api.py#L22-L26)). Run: `make test MODULE=mayan.apps.documents.tests.test_document_models` (inferred).

## Recipe 2: the authz pair (mandatory for any new endpoint)

Copy the shape at [test_document_api.py L27–L66](../../../mayan/apps/documents/tests/test_document_api.py#L27-L66) exactly: count objects → `self._clear_events()` → request via a `_request_test_X` mixin method → assert status → assert counts unchanged/changed → `self._get_test_events()` assertions. Grant with `self.grant_access(obj=..., permission=...)` (object ACL) or `self.grant_permission(...)` (role-wide) — exercising **both** tiers of [the authz machine](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) when it matters.

## Recipe 3: validation failure

```python
# Illustrative fake code: not from this repo
def test_checkout_expiration_must_be_future(self):
    with self.assertRaises(ValidationError):
        checkout = DocumentCheckout(
            document=self._test_document, user=self._test_case_user,
            expiration_datetime=now() - timedelta(days=1)
        )
        checkout.full_clean()      # clean() only runs via full_clean!
```

The lesson embedded: `clean()` is not `save()` ([checkouts/models.py L68–L72](../../../mayan/apps/checkouts/models.py#L68-L72)) — test the layer that actually enforces.

## Recipe 4: cross-tenant/ACL rejection with a second object

Create two documents; grant access to one; assert list returns exactly one and detail on the other 404s. The house mixins make N objects via repeated `self._create_test_document_stub()` calls ([test_document_api.py L68–L76](../../../mayan/apps/documents/tests/test_document_api.py#L67-L84)). This is the IDOR regression test — one per listable resource.

## Recipe 5: async side effect, tested synchronously

```python
# Illustrative fake code: not from this repo
def test_upload_task_creates_file(self):
    shared_file = SharedUploadedFile.objects.create(file=...)
    task_document_file_upload(            # call the task BODY, no broker
        document_id=self._test_document.pk,
        shared_uploaded_file_id=shared_file.pk
    )
    self.assertEqual(self._test_document.files.count(), 1)
    self.assertEqual(SharedUploadedFile.objects.count(), 0)   # cleanup contract!
```

Assert the *cleanup contract* too — that's the part that regresses ([documents/tasks.py L117–L124](../../../mayan/apps/documents/tasks.py#L117-L124)).

## Recipe 6: event-ledger assertion for a custom action

Whenever your code path commits events, pin all four fields (actor, action_object, target, verb) as at [test_document_api.py L58–L66](../../../mayan/apps/documents/tests/test_document_api.py#L42-L66). Sloppy event tests (`count() == 1` only) let attribution bugs (the `_event_actor` pop-loss class, [annotation drill 4](../04-code-reading-gym/01-annotation-drills.md)) through.

## Recipe 7: migration behavior

Shape: `MayanMigratorTestCase` with `migrate_from`/`migrate_to`, create rows in `prepare()` against the old schema, assert transformed state after. Real examples: [documents/tests/test_migrations.py](../../../mayan/apps/documents/tests/test_migrations.py). Run with migrations enabled: `make test MODULE=... SKIPMIGRATIONS=` (inferred — the default skips them, [Makefile L8–L16](../../../Makefile#L8-L16)).

## Judgment call: transaction test cases

Use `GenericTransactionDocumentTestCase` ([documents/tests/base.py L23–L28](../../../mayan/apps/documents/tests/base.py#L23-L33)) only when testing behavior that depends on real commits (signals firing post-commit, cross-connection visibility, locking) — they're slower because they truncate instead of rollback. Choosing the right base class is a review point here.

**Drill:** write recipes 2 and 4 for the hypothetical endpoint `POST /api/v4/documents/<id>/versions/<id>/modifications/` (the version-modification action). You'll hit the design question immediately: the task is fire-and-forget, so *what can the with_access test assert?* (Answer: dispatch happened — mock `apply_async` — plus a separate model test for the behavior. Write both.) *Strong:* you note the missing user feedback as a product gap, not just a test inconvenience.
