# Validation, authentication, and permissions

## Validation layers (input → invariant)

| Layer | Catches | Anchor |
| --- | --- | --- |
| URL patterns | non-numeric IDs (regex capture) | per-app `urls.py` |
| DRF serializer fields | shape, types, FK existence (`PrimaryKeyRelatedField` 400s unknown IDs) | [document_serializers.py L65–L68](../../../mayan/apps/documents/serializers/document_serializers.py#L65-L68) |
| Model `clean()` | business rules needing object context | future-dated checkout expiry ([checkouts/models.py L68–L72](../../../mayan/apps/checkouts/models.py#L68-L72)); valid workflow transition ([workflow_instance_models.py L249–L251](../../../mayan/apps/document_states/models/workflow_instance_models.py#L249-L251)) |
| Domain validators | pluggable per-metadata-type validation/parsing via dotted paths | [metadata/models.py L144–L186](../../../mayan/apps/metadata/models.py#L141-L186) |
| DB constraints | uniqueness, FK integrity, null | `unique_together`, `OneToOneField` (see [02-data-model](02-data-model-and-persistence.md)) |

Note what's *absent*: sanitization-in-depth on rendered fields relies on Django template auto-escaping; label/description are stored raw. Fine — escaping belongs at output. Say that sentence in an interview and you're ahead of most candidates.

## Authentication

Session auth for the UI (`authentication` app, plus optional OTP app), token auth for the API (`/api/v4/auth/token/obtain/`, [rest_api/urls.py L12–L16](../../../mayan/apps/rest_api/urls.py#L12-L16)). django-stronghold is in the requirements ([requirements/base.txt](../../../requirements/base.txt)) — login-required-by-default posture. Anonymous users get `queryset.none()` at the ACL layer regardless ([acls/managers.py L269–L270](../../../mayan/apps/acls/managers.py#L268-L271)) — defense in depth.

## Authorization: the three-tier machine

**Tier 1 — capability check (role-wide).** `Permission.check_user_permissions` → `StoredPermission.user_has_this`: superuser/staff always pass; otherwise any of the user's groups' roles must hold the permission ([permissions/models.py L193–L216](../../../mayan/apps/permissions/models.py#L193-L216)). Used raw for object-less views via `ViewPermissionCheckViewMixin` ([views/mixins.py L629–L649](../../../mayan/apps/views/mixins.py#L629-L649)).

**Tier 2 — object ACLs.** `AccessControlList` rows grant (permissions × role) on one object. `restrict_queryset` merges tier 1 (whole queryset if role-wide grant) with tier 2 Q-filters ([acls/managers.py L268–L294](../../../mayan/apps/acls/managers.py#L268-L294)).

**Tier 3 — inheritance.** Grants flow along registered edges: DocumentType→Document, Document→DocumentFile, etc. `_get_acl_filters` walks `ModelPermission.get_inheritances` recursively, including GenericFK edges with type casts ([acls/managers.py L31–L231](../../../mayan/apps/acls/managers.py#L31-L231) — read the 7-case comment at [L40–L49](../../../mayan/apps/acls/managers.py#L40-L49)). Grant `document_view` on a *type* once instead of N documents: administration scales by containers.

Enforcement points to memorize: API filter backend ([rest_api/filters.py L6–L24](../../../mayan/apps/rest_api/filters.py#L6-L24)), API method→permission maps on each view ([document_api_views.py L32–L37](../../../mayan/apps/documents/api_views/document_api_views.py#L23-L44)), UI `RestrictedQuerysetViewMixin` ([views/mixins.py L549–L588](../../../mayan/apps/views/mixins.py#L549-L588)), imperative `check_access` for single objects ([acls/managers.py L233–L266](../../../mayan/apps/acls/managers.py#L233-L266)), and **links/menus** — navigation itself resolves permissions so users don't see buttons they can't use (the `navigation` app).

## IDOR and the 404 posture

Denial manifests as *absence*: filtered queryset → `get_object_or_404` → *404, not 403*. Verified by the house test convention — the no-permission case asserts `HTTP_404_NOT_FOUND` ([test_document_api.py L67–L84](../../../mayan/apps/documents/tests/test_document_api.py#L67-L84)). Existence-hiding beats information leaks; the cost is support confusion ("it says not found but it exists!"). Both halves of that sentence are interview gold.

## The edge cases that make you mid-level

1. **Authorization asymmetry (verified, test-enshrined).** Creating a document demands `permission_document_create` **on the document type** ([document_api_views.py L119–L130](../../../mayan/apps/documents/api_views/document_api_views.py#L119-L130)). *Changing* a document's type demands only `properties_edit` on the document — the target type's queryset is unrestricted ([document_serializers.py L65–L68](../../../mayan/apps/documents/serializers/document_serializers.py#L65-L68)), and the test grants nothing on the target type yet expects 200 ([test_document_api.py L85–L112](../../../mayan/apps/documents/tests/test_document_api.py#L85-L112)). Consequence: a user who can edit one document can move it into any type — including types with aggressive auto-delete retention ([queues L50–L69](../../../mayan/apps/documents/queues.py#L50-L69)). Bug or decision? The test says decision; the retention interaction says it deserves an issue. This exact discussion is [review kata 6](../04-code-reading-gym/04-review-katas.md) and a mid-level ticket.
2. **Staff bypass breadth.** `is_staff` short-circuits *all* object ACLs ([permissions/models.py L200–L205](../../../mayan/apps/permissions/models.py#L193-L216)) — staff ≠ superuser in Django elsewhere, so this elevates a commonly-granted flag. Audit who has it.
3. **Trashed-but-visible.** Several APIs intentionally operate on trashed docs (tests: `test_trashed_document_*_with_access`, [test_document_api.py L114+](../../../mayan/apps/documents/tests/test_document_api.py#L114-L137)) while `valid` hides them from lists — permission to see trash is implicit in having had access. Subtle; know it exists.
4. **The `Q` truthiness trap.** Empty ACL subquery + AND-reduction would produce a wrongly-empty result; the guard comment at [acls/managers.py L119–L123](../../../mayan/apps/acls/managers.py#L119-L123) prevents case-2 filters from being added when empty. The kind of line you screenshot for "what a senior checks."

## What a junior misses vs what a senior checks

Junior: "the view has a permission, we're fine." Senior checks: which *queryset* feeds `get_object` (objects vs valid)? Is the permission on the right *object* (document vs type vs both ends of a relation)? What do **writes to related objects** require (the change-type case)? Do list, detail, and *action* views agree? Is the bypass population (staff/superuser) what the customer expects? Are machine paths (watch folder, workflow actions) exempt on purpose?

**Drill:** map the full decision tree for `GET /api/v4/documents/55/` for four users: superuser; staff; role-holder via type ACL; nobody. Write the SQL-ish shape of each outcome. *Strong:* your tree includes the anonymous short-circuit and names the exact lines where each branch exits ([acls/managers.py L268–L294](../../../mayan/apps/acls/managers.py#L268-L294)).

**Interview angle:** "Design multi-tenant authorization" / "What's IDOR and how do you prevent it structurally?" — answer: filter-at-queryset + inheritance + 404 posture + tests per endpoint pair. Cards [08/03 Q6–Q8](../08-interview-prep/03-api-and-data-modeling-questions.md); security checklist [05/05](../05-quality-engineering/05-security-checklist.md).
