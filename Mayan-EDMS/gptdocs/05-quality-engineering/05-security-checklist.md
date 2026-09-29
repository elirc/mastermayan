# Security checklist

- Authentication required where intended; session/cookie settings reviewed.
- Object authorization and list filtering both enforced: [permission adapter](../../mayan/apps/rest_api/permissions.py#L9-L59).
- Parent/child IDs scoped to prevent IDOR: [file querysets](../../mayan/apps/documents/api_views/document_file_api_views.py#L89-L120).
- Input schema, file size, filename, actual content, archive expansion, and MIME distrust addressed.
- XSS: templates escape by default; review explicit safe-marking and rich content.
- CSRF: required for cookie-authenticated mutations; API auth mode documented.
- SSRF: source/webhook URL backends validate scheme, host, redirects, and private networks.
- Injection: ORM parameterization preserved; shell/template/search queries reviewed.
- Downloads use safe content disposition and authorization; uploads are malware-scanned if required.
- Secrets stay outside repo/logs; keys rotate; dependencies scanned.
- Webhooks/signatures use replay windows and constant-time comparison.
- Rate limits/quotas protect uploads, OCR, login, search, and expensive exports.
- Audit events cannot leak sensitive payloads and have retention/access controls.

Pre-merge proof: negative permission tests, wrong-parent test, boundary validation, dependency change review, logging review, and abuse/capacity case.
