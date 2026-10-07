# Review Katas

## Kata 1: "Simplify upload permission check"
**Author intent:** reduce duplicated ACL code.
**Fake diff summary:**
- removes `restrict_queryset`
- loads `DocumentType` directly by pk
**Files this resembles:**
- [`document_api_views.py`](../../../mayan/apps/documents/api_views/document_api_views.py#L119-L129)
**Your task:** Review this PR.
**Expected findings:**
Blocking:
- introduces IDOR-style document-type access
Important:
- loses consistency with existing upload paths
Optional:
- helper extraction could still be worthwhile if it preserves authorization
**Good review comment example:**
> This gets smaller, but it also removes the ACL-filtered queryset and turns a permissioned lookup into a raw PK fetch. Can we keep the existing security boundary and extract a helper instead?

## Kata 2: "Move file metadata processing into request path"
**Author intent:** make metadata visible immediately after upload.
**Fake diff summary:**
- calls metadata driver directly from request view
- removes async task submission
**Files this resembles:**
- [`mayan/apps/file_metadata/methods.py`](../../../mayan/apps/file_metadata/methods.py#L14-L28)
- [`mayan/apps/file_metadata/tasks.py`](../../../mayan/apps/file_metadata/tasks.py#L17-L48)
**Your task:** Review this PR.
**Expected findings:**
Blocking:
- latency and failure blast radius increase
- bypasses per-file lock semantics
Optional:
- consider a "metadata pending" UX instead of synchronous work
**Good review comment example:**
> I like the product goal, but this moves previously async, lock-guarded work into the request path. Can we keep the task boundary and surface "metadata pending" instead?

## Kata 3: "Replace delegated click handlers with direct bindings"
**Author intent:** simplify front-end code.
**Fake diff summary:**
- replaces delegated handlers with direct bindings on page load
**Files this resembles:**
- [`mayan/apps/appearance/static/appearance/js/partial_navigation.js`](../../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L273-L280)
**Your task:** Review this PR.
**Expected findings:**
Blocking:
- partial navigation will break after DOM replacement
Important:
- loses compatibility with AJAX-loaded content
**Good review comment example:**
> Because `#ajax-content` is replaced repeatedly, direct bindings will only cover the initial DOM. The delegated pattern is carrying a real runtime requirement here.

## Kata 4: "Catch all OCR exceptions and continue"
**Author intent:** keep OCR queue moving.
**Fake diff summary:**
- broad `except Exception: return`
**Files this resembles:**
- [`mayan/apps/ocr/tasks.py`](../../../mayan/apps/ocr/tasks.py#L75-L89)
**Your task:** Review this PR.
**Expected findings:**
Blocking:
- hides failures and harms observability
Important:
- breaks retry semantics for transient failures
**Good review comment example:**
> This makes the queue quieter, but it also removes the signal operators need when OCR is degraded. Can we preserve retries for transient failures and keep explicit error logging?

## Kata 5: "Add document checkout by bulk SQL update"
**Author intent:** speed up a maintenance operation.
**Fake diff summary:**
- bypasses model `save()`
- inserts checkout rows directly
**Files this resembles:**
- [`mayan/apps/checkouts/models.py`](../../../mayan/apps/checkouts/models.py#L100-L116)
**Your task:** Review this PR.
**Expected findings:**
Blocking:
- may allow duplicate checkouts under application rules
Important:
- bypasses save-time invariant and event semantics
**Good review comment example:**
> The direct write is faster, but it also bypasses the invariant and audit events encoded in `DocumentCheckout.save()`. If we need bulk behavior, we should design a path that preserves those contracts.

## Kata 6: "Emit source callback before file save"
**Author intent:** allow custom source behavior earlier.
**Fake diff summary:**
- callback runs before `DocumentFile` is fully initialized
**Files this resembles:**
- [`mayan/apps/sources/models.py`](../../../mayan/apps/sources/models.py#L45-L65)
**Your task:** Review this PR.
**Expected findings:**
Blocking:
- callback assumes `document_file.pages.all()` exists
Important:
- inverts current dependency ordering
**Good review comment example:**
> The callback currently copies transformations onto already-created pages. Running it earlier would break that assumption and likely surprise source backends that expect a stable `DocumentFile`.

## Kata 7: "Skip temp upload deletion for easier debugging"
**Author intent:** preserve failure evidence.
**Fake diff summary:**
- removes `shared_uploaded_file.delete()`
**Files this resembles:**
- [`mayan/apps/sources/tasks.py`](../../../mayan/apps/sources/tasks.py#L95-L103)
**Your task:** Review this PR.
**Expected findings:**
Important:
- likely orphan growth unless paired with retention cleanup
Optional:
- consider a sweeper command or flagged retention mode
**Good review comment example:**
> Keeping failed uploads can help debugging, but deleting nothing by default creates a storage-retention problem. Could we pair this with bounded retention or a sweeper command?

## Kata 8: "Open password reset to anonymous and disable CSRF for convenience"
**Author intent:** ease support burden.
**Fake diff summary:**
- strips CSRF and weakens auth-flow protections
**Files this resembles:**
- [`mayan/apps/authentication/views/authentication_views.py`](../../../mayan/apps/authentication/views/authentication_views.py#L277-L302)
- [`mayan/settings/base.py`](../../../mayan/settings/base.py#L108-L115)
**Your task:** Review this PR.
**Expected findings:**
Blocking:
- auth boundary regression
Important:
- weakens secure-by-default posture
**Good review comment example:**
> This would make the flow easier to reach, but at the cost of weakening the repo’s secure defaults. Can we solve the support issue without disabling CSRF or changing the broader auth assumptions?
