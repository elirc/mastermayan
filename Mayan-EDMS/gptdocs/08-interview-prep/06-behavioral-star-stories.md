# Behavioral STAR worksheets

Use these as worksheets. Replace proposed outcomes with evidence from work you actually performed; studying a repo is not the same as shipping a contribution.

## Story: learned a complex codebase quickly
Prompts: ambiguity, learning. Source: fast-track trace. Situation/Task: unfamiliar Django EDMS; build a reliable map. Action: traced API→permission→task→model→test using anchors. Result: fill in time saved or teach-back score. Senior signal: separated claims from hypotheses. Resume bullet: “Mapped a multi-app Django/Celery system and documented cross-layer upload/security flows.”

## Story: found a security boundary
Prompts: risk, attention to detail. Source: Ticket 1. Action: explain parent scoping and wrong-parent test from [the view](../../mayan/apps/documents/api_views/document_file_api_views.py#L89-L120). Result: only claim test/PR outcome if completed. Senior signal: threat without alarmism. Resume bullet: “Strengthened nested-resource authorization coverage against IDOR-style access.”

## Story: debugged asynchronous work
Prompts: difficult bug, method. Source: upload pending scenario. Action: split request/enqueue/worker/storage and use cheapest probes. Result: record measured resolution. Senior signal: terminal visibility/idempotency. Resume bullet: “Narrowed an async processing failure across API, queue, worker, and storage boundaries.”

## Story: improved reliability
Prompts: tradeoff, ownership. Source: staging orphan ticket/project. Action: failure injection, contract decision, bounded cleanup, metrics. Result: orphan rate or test coverage. Senior signal: compensation versus fake atomicity. Resume bullet: “Designed failure-safe staging cleanup with reconciliation and observability.”

## Story: handled disagreement in review
Prompts: conflict. Source: parent-scope review kata. Action: cite invariant/test, distinguish blocker from preference, invite alternative preserving outcome. Result: agreed design. Senior signal: kind specificity. Resume bullet: “Resolved a security-sensitive review through evidence and a focused regression test.”

## Story: measured before optimizing
Prompts: performance. Source: ACL benchmark ticket. Action: representative fixtures, query count/plan, p95, correctness guard. Result: actual metric. Senior signal: rejected premature rewrite. Resume bullet: “Benchmarked authorization query paths and prioritized evidence-backed optimization.”

## Story: designed a migration
Prompts: technical tradeoff, planning. Source: operation status project. Action: expand/backfill/dual compatibility/contract/rollback. Result: rehearsal artifact or shipped outcome. Senior signal: old worker payloads. Resume bullet: “Planned a backward-compatible async-operation schema rollout with rollback gates.”

## Story: made failures visible
Prompts: initiative, user focus. Source: queue metrics or operation status. Action: define SLO/signals/dashboard/runbook. Result: time-to-detect improvement if real. Senior signal: bounded-cardinality labels and actionable alert. Resume bullet: “Added end-to-end visibility for queued document processing and terminal failures.”

Rehearsal check for every story: under two minutes, first-person actions, concrete evidence, tradeoff and risk, no inflated authorship, ends with impact/learning.
