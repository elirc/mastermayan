# Systematic Debugging

## Method

1. Reproduce.
2. Narrow the layer.
3. Form a cheap hypothesis.
4. Probe with the lightest signal available.
5. Fix root cause.
6. Add regression coverage.

## Scenario: Upload accepted but document never appears
**Reproduction:** POST upload returns `202`, but no document shows up.
**First question:** Is the bug in request-side enqueue or worker-side processing?
**Narrowing path:**
1. Check request path in [`web_form_backends.py`](../../../mayan/apps/sources/source_backends/web_form_backends.py#L57-L72).
2. Check worker path in [`sources/tasks.py`](../../../mayan/apps/sources/tasks.py#L64-L103).
3. Inspect temp upload cleanup assumptions.
**Useful probes:**
- log whether `SharedUploadedFile` row still exists
- inspect worker logs for `OperationalError`
**Likely root causes:**
- worker not running
- missing DB dependency
- callback or downstream save failure
**Regression test to add:**
- failure-path test that asserts temp uploads are eventually cleaned or reported
**Senior lesson:** async acknowledgment without strong visibility is an operability risk.

## Scenario: User can see a document type they should not upload into
**Reproduction:** User without create access still sees or can target a document type.
**First question:** Was the queryset ACL-filtered before lookup?
**Narrowing path:**
1. Inspect upload API and source backend permission gates.
2. Search for raw `DocumentType.objects.get(...)`.
3. Compare with the standard `restrict_queryset` pattern.
**Useful probes:**
- grep for `document_type_id`
- permission regression tests in upload suites
**Likely root causes:**
- raw `DocumentType.objects.get(pk=...)`
**Regression test to add:**
- denied upload attempt via API and via source backend
**Senior lesson:** authorization regressions often enter during innocent-looking cleanup refactors.

## Scenario: OCR jobs pile up
**Reproduction:** OCR queue grows while documents remain unprocessed.
**First question:** Are page tasks retrying due to cache or lock failures?
**Narrowing path:**
1. Inspect page-task retry branches.
2. Verify cached page images exist.
3. Inspect lock and DB health.
**Useful probes:**
- inspect retryable exceptions in [`ocr/tasks.py`](../../../mayan/apps/ocr/tasks.py#L80-L89)
**Likely root causes:**
- cache image missing
- lock contention
- DB operational error
**Regression test to add:**
- retry-path test for a missing cache file
**Senior lesson:** fan-out systems need per-unit visibility, not just final failure counts.

## Scenario: Document checkout says already checked out unexpectedly
**Reproduction:** checkout attempt fails even though the user expects the document to be free.
**First question:** Is there an existing row or a stale state assumption?
**Narrowing path:**
1. Inspect the existing checkout row.
2. Verify expiration expectations.
3. Check for any path bypassing model `save()`.
**Useful probes:**
- inspect `DocumentCheckout.save()` invariant in [`checkouts/models.py`](../../../mayan/apps/checkouts/models.py#L100-L116)
**Likely root causes:**
- valid existing checkout
- concurrent request race
- direct write bypassing expected semantics
**Regression test to add:**
- repeated checkout attempt test
**Senior lesson:** schema shape and business semantics are related but not identical.

## Scenario: Browser navigation loads stale or broken content
**Reproduction:** fast navigation leaves mismatched or stale content in the main pane.
**First question:** Is this a backend partial-render problem or a front-end AJAX race?
**Narrowing path:**
1. Verify request cancellation behavior.
2. Inspect whether latest-response-wins is preserved.
3. Check whether event binding survived DOM replacement.
**Useful probes:**
- inspect request cancellation in [`partial_navigation.js`](../../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L107-L115)
- inspect error rendering in [`partial_navigation.js`](../../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L201-L247)
**Likely root causes:**
- race between responses
- delegated handler regression
- backend error surfaced as generic content replacement
**Regression test to add:**
- browser or manual test with rapid repeated clicks
**Senior lesson:** legacy UI code can still encode good concurrency instincts worth preserving.
