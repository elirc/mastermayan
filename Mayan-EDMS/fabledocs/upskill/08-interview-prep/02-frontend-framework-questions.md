# Frontend framework questions — question cards

11 cards. Mayan's UI is **server-rendered Django templates + Bootstrap + jQuery** ([appearance/dependencies.py](../../../mayan/apps/appearance/dependencies.py#L38-L51)), not React. That's an asset in interviews: you can discuss *why* a document-management app chose no SPA, and contrast every React concept with the server-rendered alternative. Where a card is React-specific, the anchor is "conceptual" but the *architectural* comparison uses real Mayan decisions.

---

## Q1: Server-rendered vs SPA — when is each right?
**Testing:** architectural judgment, not framework loyalty.
**Repo anchor:** the whole UI ([document_views.py](../../../mayan/apps/documents/views/document_views.py#L34-L78) renders server-side; state lives in URL + DB).
**Junior:** "SPAs are more modern."
**Mid adds:** server-rendering wins for content-heavy, SEO/print-friendly, low-interactivity apps with strong server authorization; SPAs win for rich client interactivity and offline. Mayan is CRUD-over-documents with per-object authz — server-rendered is the *right* call, not legacy debt.
**Senior adds:** the deciding axis is where state and trust live; Mayan keeps both on the server so the client is a thin view — which means no client-side authz to get wrong (a whole vuln class avoided). The cost is a page-load-per-interaction UX ceiling.
**Follow-up:** how would you add one rich interaction (drag-to-reorder pages) without adopting a SPA? (a scoped JS component + an API endpoint — the version-remap API already exists).

## Q2: Rendering model — React render/commit vs template render
**Anchor:** conceptual; contrast with Mayan's "render once per request."
**Mid adds:** React diffs a virtual DOM then commits; a template renders a full HTML string server-side with no reconciliation. React's re-render triggers (state/props) have no analog — Mayan re-fetches the page.
**Senior adds:** the tradeoff is interactivity vs simplicity; React's cost is the reconciliation mental model (keys, memoization, stale closures) that Mayan simply doesn't have.

## Q3: State placement — where should state live?
**Anchor:** Mayan: URL query params (sorting `?_ordering=`, [views/mixins.py L610](../../../mayan/apps/views/mixins.py#L590-L611)) + DB; no client store.
**Junior:** "in useState."
**Mid adds:** the hierarchy — server (source of truth), URL (shareable view state), component (ephemeral UI). Mayan pushes almost everything to server+URL; that's why a bookmarked filtered list *just works*.
**Senior adds:** lifting state up vs colocating; the URL as state is underused in SPAs and is Mayan's default — sortable/filterable lists are reconstructable from the URL alone.
**Follow-up:** what state should *never* go in the URL? (secrets, huge blobs).

## Q4: Data fetching — REST vs the coupling to a server-rendered app
**Anchor:** Mayan exposes both server-rendered pages *and* a full REST API ([rest_api/](../../../mayan/apps/rest_api/urls.py) over the same models/serializers).
**Mid adds:** the API and UI share the authorization layer and serializers — one source of truth for shape and access. In a React app you'd hit that REST API with React Query; the caching/staleness concerns you'd add on the client are handled server-side here.
**Senior adds:** the risk of dual surfaces (UI form path vs API path diverging on authz — the change-type UI-vs-API gap, [06/M2](../06-contribution-practice/02-mid-level-feature-tickets.md)); every action needs coverage on *both* paths.

## Q5: Effects and lifecycle — the stale closure bug
**Anchor:** conceptual; the Python twin is the loop-capture bug, solved by [handler factories](../../../mayan/apps/dynamic_search/handlers.py#L56-L81).
**Mid adds:** `useEffect` deps that capture stale values cause bugs identical in spirit to a closure capturing a loop variable; the fix (fresh capture / correct deps) is the same idea Mayan uses in its factories.
**Senior adds:** cleanup functions, effect ordering, why exhaustive-deps lint exists.

## Q6: Lists and keys / identity
**Anchor:** Mayan's `natural_key` ([document_models.py L276–L278](../../../mayan/apps/documents/models/document_models.py#L276-L278)) — stable identity across systems, the server-side analog of React keys.
**Mid adds:** keys let React match elements across renders; unstable keys (index) cause state bleakage — same principle as choosing a stable DB identity (UUID vs autoincrement) for merges.
**Senior adds:** identity is a design decision at every layer; a bad key in React ≈ a natural key that isn't actually unique.

## Q7: Performance — memoization vs server-side query cost
**Anchor:** Mayan's per-row property cost (`file_latest`, [document_models.py L185–L187](../../../mayan/apps/documents/models/document_models.py#L185-L187)) — an N+1 that's the server analog of an unmemoized expensive render.
**Mid adds:** `useMemo`/`React.memo` avoid recomputing on the client; Mayan avoids per-row queries via prefetch/annotate. Same instinct: don't repeat expensive work per item.
**Senior adds:** measure first (profiler / `assertNumQueries`); premature memoization has a cost too.

## Q8: Accessibility and forms
**Anchor:** Django forms + widget tweaks ([requirements/base.txt](../../../requirements/base.txt) `django-widget-tweaks`); server-side validation errors rendered inline.
**Mid adds:** server-rendered forms get progressive enhancement and work without JS; accessibility is HTML-semantics-first. Client validation is a UX nicety, server validation is the contract ([checkouts clean()](../../../mayan/apps/checkouts/models.py#L68-L72)).
**Senior adds:** never trust client validation; label/aria semantics; error summary placement.

## Q9: Optimistic updates and consistency
**Anchor:** Mayan is the *opposite* — async upload returns before the file exists ([Flow 1](../01-codebase-cartography/05-key-flows.md)), so the UI shows a stub/pending state.
**Mid adds:** optimistic UI assumes success and rolls back on failure; Mayan instead surfaces pending state honestly (the document is a stub until the worker finishes). Both are answers to "the write isn't done yet."
**Senior adds:** optimistic updates need a rollback path and reconciliation; the server-rendered honest-pending approach trades snappiness for simplicity and truth.

## Q10: Bundling and dependency delivery
**Anchor:** Mayan *vendors* JS libs with versions/hashes ([appearance/dependencies.py](../../../mayan/apps/appearance/dependencies.py#L38-L51)) instead of bundling.
**Mid adds:** no webpack/vite step; libraries are downloaded and served — a deliberate offline/self-hosted-friendly choice, the anti-pattern for a modern SPA but sensible for on-prem document management.
**Senior adds:** the tradeoff (no tree-shaking/code-splitting vs zero build complexity and air-gap friendliness).

## Q11: Component reuse — mixins/inheritance vs composition
**Anchor:** Mayan's view mixins ([views/mixins.py](../../../mayan/apps/views/mixins.py#L549-L649)) compose behavior by inheritance/MRO.
**Mid adds:** React moved from mixins/HOCs to hooks (composition over inheritance) precisely because deep mixin chains (like Mayan's 6-mixin views) get hard to trace; the MRO *is* the composition order.
**Senior adds:** the readability cost of deep inheritance (which `get_queryset` wins?) is real; hooks/composition trade it for explicit wiring. You can critique Mayan's view stack on exactly this axis.

**Self-grade:** *Solid* — you can contrast every React concept with Mayan's server-rendered choice. *Strong* — for Q1 and Q9 you argue the architecture both ways and land a recommendation for a given product. The meta-skill: never trash server-rendering as "old" — defend it where it's right, which for a document vault is most places.
