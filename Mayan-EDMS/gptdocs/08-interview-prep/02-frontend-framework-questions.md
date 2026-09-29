# Frontend/framework questions

Mayan is primarily server-rendered Django, so answer in that model and translate to React where useful.

## Q13: Walk me through a server-rendered request lifecycle.
Round: frontend. Repo anchor: [download retrieve](../../mayan/apps/documents/api_views/document_file_api_views.py#L93-L123). Junior: route→view→response. Mid: auth, queryset, permission, representation, event. Senior: streaming, headers, storage failure, observability. Follow-up: React SPA difference.

## Q14: Forms versus API serializers—what belongs where?
Round: frontend. Repo anchor: [upload serializer](../../mayan/apps/documents/serializers/document_file_serializers.py#L71-L140). Junior: validate input. Mid: protocol-specific shape at boundary, domain invariant below both. Senior: prevent duplicated divergent rules. Drill: classify rules.

## Q15: How would you show async upload progress?
Round: frontend. Repo anchor: [202 response](../../mayan/apps/documents/api_views/document_file_api_views.py#L38-L60). Junior: spinner. Mid: operation ID, polling/SSE, success/failure/retry/cancel states. Senior: refresh recovery, accessibility, optimistic truth, expiry. Follow-up: offline. Drill: state diagram.

## Q16: What makes a list UI safe under object permissions?
Round: frontend. Repo anchor: [ACL-filtered queryset](../../mayan/apps/acls/managers.py#L268-L294). Junior: hide buttons. Mid: server filters data; UI hiding is not security. Senior: caching/count/pagination/search scope. Follow-up: stale permissions. Drill: threat model.

## Q17: How do React effects differ from server request effects?
Round: frontend. Repo anchor: conceptual—server side effect in [download event test](../../mayan/apps/documents/tests/test_document_file_api.py#L160-L186). Junior: effects run after render. Mid: React effects may repeat; server commands need idempotency too. Senior: Strict Mode, aborting fetches, event semantics. Drill: design safe analytics.

## Q18: Where should UI state live?
Round: frontend. Repo anchor: conceptual document/file/version hierarchy at [file model](../../mayan/apps/documents/models/document_file_models.py#L74-L120). Junior: component state. Mid: URL for navigation, server cache for resources, local state for ephemeral input. Senior: ownership minimizes synchronization. Follow-up: selected version. Drill: classify ten states.

## Q19: How would you prevent stale data after upload completes?
Round: frontend. Repo anchor: upload is eventually completed by [worker](../../mayan/apps/documents/tasks.py#L82-L124). Junior: refetch. Mid: invalidate document/file keys after terminal operation. Senior: scoped cache keys, races, monotonic version, push events. Drill: cache-key design.

## Q20: What accessibility concerns exist in document processing UI?
Round: frontend. Repo anchor: conceptual. Junior: labels/keyboard. Mid: progress announcements, focus management, error association, non-color status, accessible preview/download. Senior: large async workflows and recovery. Drill: audit upload states.

## Q21: When is memoization useful in React?
Round: frontend. Repo anchor: conceptual contrast to server query performance. Junior: avoid rerender. Mid: only measured expensive computation or identity-sensitive child. Senior: cache invalidation/complexity cost; fix state ownership first. Follow-up: `useMemo` guarantee. Drill: explain when not to use.

## Q22: How do you avoid leaking protected data through frontend caches?
Round: frontend. Repo anchor: identity-dependent [ACL restriction](../../mayan/apps/acls/managers.py#L268-L294). Junior: clear on logout. Mid: scope keys by identity/tenant and never trust client protection. Senior: permission changes, shared SSR cache, persistence, sensitive previews. Drill: cache threat table.
