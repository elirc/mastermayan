# Security checklist

Mapped to this codebase. "Covered" = the repo has a real defense you can point at; "Watch" = a place to verify before shipping a change. Nothing here is a claimed exploit — items marked *investigate* are static-reading suspicions.

| Threat | Status | Where |
| --- | --- | --- |
| **Broken object-level authz (IDOR)** | Covered (structural) | `restrict_queryset` filters every list/detail; denial = 404; per-endpoint test pairs ([acls/managers.py L268–L294](../../../mayan/apps/acls/managers.py#L268-L294), [test_document_api.py L67–L84](../../../mayan/apps/documents/tests/test_document_api.py#L67-L84)) |
| **Privilege via related writes** | Watch | change-type asymmetry lets doc-editors move docs into any type ([03/03 §edge cases](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)) — decision, but audit for your tenancy |
| **Over-broad bypass** | Watch | `is_staff` short-circuits all object ACLs ([permissions/models.py L200–L205](../../../mayan/apps/permissions/models.py#L193-L216)) — inventory who has staff |
| **Input validation** | Covered | DRF field validation; model `clean()`; DB constraints ([02/03 checkpoint table](../02-stack-and-language-mastery/03-type-system-and-contracts.md)) |
| **XSS** | Covered (by default) | Django template auto-escaping; labels/descriptions stored raw, escaped at render. Watch any `\|safe`/`mark_safe` in templates and rich HTML fields |
| **CSV/formula injection** | *Investigate* | event export writes `str(value)` unescaped ([events/classes.py L62–L68](../../../mayan/apps/events/classes.py#L41-L68)); a label like `=HYPERLINK(...)` executes in Excel. Low severity, real |
| **SSRF** | Watch | workflow HTTP actions POST to admin-configured URLs (document_states workflow_actions); watch-folder/staging paths are server-side FS ([watch_folder_backends.py L88–L96](../../../mayan/apps/sources/source_backends/watch_folder_backends.py#L77-L112)). Trust model: admin-only config = accepted risk; verify who can create sources/actions |
| **Path traversal (upload)** | Covered | filename reduced to `Path(...).name` before storage ([document_models.py L233](../../../mayan/apps/documents/models/document_models.py#L230-L234)) |
| **CSRF** | Covered | Django middleware for UI; API uses token auth (not cookie) so CSRF N/A for API |
| **Injection (SQL)** | Covered | ORM everywhere; the one raw-ish spot is `connection.vendor` branching in search UUID handling ([documents/search.py L12–L16](../../../mayan/apps/documents/search.py#L12-L16)) — no interpolation of user input |
| **Secrets** | Covered | env var → file → default fallback ([settings/base.py L30–L38](../../../mayan/settings/base.py#L30-L38)); no secrets in repo. The default placeholder key must be overridden in prod (deploy checklist item) |
| **Dependency risk** | Watch | 2022 pins, Django 3.2 (EOL) ([requirements/base.txt](../../../requirements/base.txt)); study artifact, not deployable as-is |
| **File uploads (content)** | Partial | MIME detected server-side ([document_file_models.py L344–L361](../../../mayan/apps/documents/models/document_file_models.py#L344-L361)); archive `expand` recurses ([document_models.py L205–L224](../../../mayan/apps/documents/models/document_models.py#L189-L251)) — zip-bomb/nested-archive DoS surface; converter runs on untrusted files (sandbox tesseract/soffice at the ops layer) |
| **Rate limiting** | Missing | no API throttle in the base classes ([rest_api/generics.py](../../../mayan/apps/rest_api/generics.py#L20-L72)); token-obtain endpoint unthrottled — brute-force surface |
| **Distributed lock integrity** | *Investigate* | file-lock `_release` catches wrong exception; module thread-lock leak on error ([file_lock.py L94–L101](../../../mayan/apps/lock_manager/backends/file_lock.py#L94-L116)) — availability, not confidentiality |
| **Notification/subscription abuse** | Watch | events fan out to subscribers synchronously in save ([events/classes.py L387–L427](../../../mayan/apps/events/classes.py#L359-L429)) — a subscription storm is a self-DoS vector |

## Pre-merge security checklist (paste into PR description)

- [ ] New list/detail endpoints go through `restrict_queryset` (base class or explicit) and have a `no_permission` → 404 test.
- [ ] Writes that touch a *related* object check permission on that object too (not just the parent).
- [ ] No new `mark_safe`/`|safe` on user-controlled data.
- [ ] No user input reaches a shell/URL/filesystem path without validation; new outbound URLs are admin-config only or allow-listed.
- [ ] New settings don't log secrets; no secret defaults that "work" in prod.
- [ ] New uploads bound recursion/size; converters run on the sandboxed worker tier.
- [ ] New public API endpoints considered for throttling.
- [ ] Events fired match intended audit policy (assert all four fields in tests).

**Drill:** pick the CSV-injection item. Write the fix (prefix risky leading chars per OWASP) *and* argue whether it belongs in the exporter or is out of scope (the data is admin-exported, opened in the admin's Excel — is that a vuln or a documentation note?). *Strong:* your answer distinguishes "our bug" from "spreadsheet behavior" and still ships the one-line defense because it's cheap.

**Interview angle:** "How do you prevent IDOR?" → the structural answer (filter-at-queryset). "Walk me through securing a file-upload feature." → path-traversal neutralization + MIME detection + archive-bomb bounds + sandboxed conversion, all with anchors. Cards [08/03 Q6–Q8, Q18](../08-interview-prep/03-api-and-data-modeling-questions.md).
