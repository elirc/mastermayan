# Pattern catalog

Sixteen cards. The goal is **recognition** — you should be able to point at any of these in a codebase you've never seen and name the problem it solves plus one failure mode. Format per card: problem → shape → real anchors → why it works → failure modes → when (not) to use → interview angle → drill.

---

## Pattern 1: Self-registering modules

**Problem:** compose 57 optional apps into one site without a central wiring file that everyone edits (merge-conflict magnet, coupling magnet).
**Shape:** each module's init hook appends its routes/permissions/queues to global registries.
**Real example:** `MayanAppConfig.configure_urls` appends to the initially-empty urlconf ([common/apps.py L27–L120](../../../mayan/apps/common/apps.py#L27-L120), [mayan/urls/base.py](../../../mayan/urls/base.py)). Second: queue self-registration per app ([documents/queues.py](../../../mayan/apps/documents/queues.py)).
**Why it works:** adding an app = adding it to `INSTALLED_APPS`; deletion is equally clean.
**Failure modes:** wiring is invisible to grep; import order becomes semantics ([settings/base.py L42–L48](../../../mayan/settings/base.py#L42-L48)); a bad app breaks *startup*, not its own page.
**Use when:** plugin ecosystems. **Avoid when:** a 5-module app — explicitness wins.
**Interview angle:** "how would you build a plugin system?"
**Drill:** find where the `tags` app registers its ACL permissions without opening more than 2 files.

## Pattern 2: Class registry via `_registry` dict

**Problem:** look up domain classes (event types, search models, permissions) by stable string ID at runtime.
**Shape:** class attribute dict; `__init__` writes `self` into it; classmethod `get(id)` reads.
**Real example:** [EventType, events/classes.py L323–L354](../../../mayan/apps/events/classes.py#L323-L354). Second: [PermissionNamespace/Permission, permissions/classes.py L16–L129](../../../mayan/apps/permissions/classes.py#L16-L49).
**Why it works:** IDs (`documents.document_created`) are storable in DB rows and stable across processes.
**Failure modes:** global mutable state (test pollution); no per-instance variation; import-time dependency ordering.
**Interview angle:** singleton/registry tradeoffs; "why not DI?"
**Drill:** add a hypothetical `document_starred` event on paper: which 3 lines in which files?

## Pattern 3: Authorization as queryset restriction

**Problem:** per-object permissions without `if can_see(obj)` sprinkled everywhere (and without loading forbidden rows at all).
**Shape:** one function turns (queryset, permission, user) into a filtered queryset; every list/detail passes through it.
**Real example:** [restrict_queryset, acls/managers.py L268–L294](../../../mayan/apps/acls/managers.py#L268-L294). Second: its API and UI consumers ([rest_api/filters.py L6–L16](../../../mayan/apps/rest_api/filters.py#L6-L24), [views/mixins.py L549–L588](../../../mayan/apps/views/mixins.py#L549-L588)).
**Why it works:** default-deny; denial = 404; single choke point to audit.
**Failure modes:** N subqueries on hot lists; code that bypasses managers (raw SQL, `objects.all()` in a new view) silently skips it.
**Interview angle:** IDOR prevention *structurally* — the answer is this card.
**Drill:** write the fake diff that introduces an IDOR here (one changed line in a view) — then the test that catches it.

## Pattern 4: Permission inheritance graph

**Problem:** granting per-document is administratively hopeless at scale.
**Shape:** registered parent-edges; the ACL filter recursively ORs parent grants.
**Real example:** [_get_acl_filters cases 4–6, acls/managers.py L125–L194](../../../mayan/apps/acls/managers.py#L125-L194).
**Why it works:** grant on containers (type, cabinet), objects inherit.
**Failure modes:** recursion cost; GenericFK edges need type casts ([L95–L106](../../../mayan/apps/acls/managers.py#L95-L107)); reasoning about *effective* access requires a tool, not eyes ([get_inherited_permissions, L296–L310](../../../mayan/apps/acls/managers.py#L296-L310)).
**Interview angle:** RBAC vs ReBAC; this is a hand-rolled ReBAC-lite.
**Drill:** list every model a DocumentType grant can reach, using only `ModelPermission.register_inheritance` call sites.

## Pattern 5: Audit events via method decorator

**Problem:** every mutation must produce an audit record with actor attribution, without 500 copies of `log(...)`.
**Shape:** decorator wraps `save`/`delete`; actor smuggled via instance attribute; event committed after the method.
**Real example:** [method_event, events/decorators.py L8–L33](../../../mayan/apps/events/decorators.py#L8-L33) + [Document.save, document_models.py L292–L316](../../../mayan/apps/documents/models/document_models.py#L292-L316). Second: dynamic event type chosen at runtime in checkout delete ([checkouts/models.py L74–L87](../../../mayan/apps/checkouts/models.py#L74-L87)).
**Why it works:** the mutation site can't forget to audit.
**Failure modes:** hidden `_event_actor` parameter; `_event_ignore` escape hatch scattered ([document_models.py L150](../../../mayan/apps/documents/models/document_models.py#L146-L152)); sync fan-out inside save (see [03/04](04-side-effects-async-and-reliability.md)).
**Interview angle:** AOP/cross-cutting concerns; compare with middleware.
**Drill:** trace which event fires when a checkout expires vs when a user checks in ([checkouts/models.py L74–L87](../../../mayan/apps/checkouts/models.py#L74-L87)).

## Pattern 6: Soft delete with manager taxonomy + proxies

**Problem:** deletion must be reversible, policy-driven, and invisible to normal queries.
**Shape:** flag + flip-first `delete()`; `valid`/`trash` managers; proxy model for the trash UI.
**Real example:** [Document.delete, document_models.py L142–L163](../../../mayan/apps/documents/models/document_models.py#L142-L163); managers at [L106–L108](../../../mayan/apps/documents/models/document_models.py#L106-L108).
**Failure modes:** any code using the wrong manager; uniqueness constraints colliding with trashed rows; forgotten cascade semantics on hard delete.
**Interview angle:** "how do you do soft delete without WHERE-clause litter?"
**Drill:** Flow 8's manager drill.

## Pattern 7: Staging-table handoff (SharedUploadedFile)

**Problem:** move big uploads from request to worker without serializing bytes into the broker.
**Shape:** persist blob + row; enqueue the row ID; consumer deletes on every exit path.
**Real example:** [document_serializers.py L74–L90](../../../mayan/apps/documents/serializers/document_serializers.py#L74-L90) → [documents/tasks.py L52–L124](../../../mayan/apps/documents/tasks.py#L52-L124). Second: watch folder staging ([watch_folder_backends.py L100–L112](../../../mayan/apps/sources/source_backends/watch_folder_backends.py#L100-L112)).
**Why it works:** broker stays small/fast; staging survives crashes; cleanup is explicit.
**Failure modes:** leak-on-missed-exit-path; staging becomes an unbounded queue if workers stall.
**Interview angle:** "why not put the file in the message?" — answer with this card.
**Drill:** count the `shared_uploaded_file.delete()` call sites in one task and match each to a failure class.

## Pattern 8: IDs-across-the-queue + late rebinding

**Problem:** queued messages outlive process memory and code versions.
**Shape:** primitive kwargs; worker re-fetches with `apps.get_model`.
**Real example:** [documents/tasks.py L26–L36](../../../mayan/apps/documents/tasks.py#L26-L36). Second: search tasks ([dynamic_search/tasks.py L46–L63](../../../mayan/apps/dynamic_search/tasks.py#L41-L63)).
**Failure modes:** row deleted before the worker runs (the deindex *investigate* in Flow 7); kwargs become an unversioned contract.
**Interview angle:** message design; deploy-window compatibility.
**Drill:** find one task whose kwargs changed would strand in-flight messages — and the additive alternative.

## Pattern 9: Pluggable engine backends (dotted-path strategy)

**Problem:** search/locking/storage/OCR engines must be swappable per deployment.
**Shape:** abstract base + `get_backend()` reading a setting containing a dotted path; backends imported lazily.
**Real example:** [LockingBackend](../../../mayan/apps/lock_manager/backends/base.py) with three implementations ([file_lock](../../../mayan/apps/lock_manager/backends/file_lock.py), [model_lock](../../../mayan/apps/lock_manager/backends/model_lock.py), redis_lock). Second: duplicates backends ([duplicate_backends.py L8–L28](../../../mayan/apps/duplicates/duplicate_backends.py#L8-L28)).
**Why it works:** ops chooses the engine in env config ([docker-compose.yml L12–L13](../../../docker/docker-compose.yml#L3-L16)).
**Failure modes:** the interface is only as honest as its weakest backend (file lock is host-local — correctness depends on deployment shape!); capability drift between backends.
**Interview angle:** strategy pattern at ops scale; "how would you support both Whoosh and Elasticsearch?"
**Drill:** list two behaviors Redis locks have that file locks cannot honor across hosts.

## Pattern 10: Derived state with explicit invariant methods

**Problem:** fields computable from other state (checksum, size, page count, active flag) must stay consistent.
**Shape:** `X_update()` methods, batched in a transaction at the right lifecycle moment.
**Real example:** [checksum_update/mimetype_update/size_update/page_count_update, document_file_models.py L186–L213, L344–L361, L389–L414, L514–L525](../../../mayan/apps/documents/models/document_file_models.py#L186-L213), orchestrated at [L466–L472](../../../mayan/apps/documents/models/document_file_models.py#L466-L472). Second: version `active_set` ([document_version_models.py L95–L101](../../../mayan/apps/documents/models/document_version_models.py#L95-L101)).
**Failure modes:** anyone mutating the file outside the pipeline desyncs everything (the `exists()` doc-comment admits storage can desync, [L248–L258](../../../mayan/apps/documents/models/document_file_models.py#L248-L258)).
**Interview angle:** derived data — recompute vs store; invariants and their enforcement point.
**Drill:** which invariant would a direct SQL `UPDATE document_file SET file=...` break, and what detects it?

## Pattern 11: Fan-out/fan-in (chord)

**Problem:** parallelize per-page work, then run a completion step exactly once.
**Real example:** [ocr/tasks.py L31–L48](../../../mayan/apps/ocr/tasks.py#L17-L49).
**Failure modes:** requires result backend; one poisoned unit stalls the join; completion semantics on retry.
**Interview angle:** Promise.all analogy + differences (Flow 5 drill).
**Drill:** Flow 5's.

## Pattern 12: Per-entity error logs as user-facing observability

**Problem:** background failures must reach the *owner of the data*, not just ops logs.
**Shape:** generic `error_log` relation on sources/versions; tasks write there; success clears.
**Real example:** [sources/tasks.py L39–L53](../../../mayan/apps/sources/tasks.py#L39-L53); OCR error rows ([ocr/tasks.py L44–L48](../../../mayan/apps/ocr/tasks.py#L44-L48)) and log clearing on finish ([L105](../../../mayan/apps/ocr/tasks.py#L92-L117)).
**Why it works:** admins see "why didn't my folder import" in the UI.
**Failure modes:** clearing on success can erase evidence of intermittent failures; unbounded growth without pruning.
**Interview angle:** "how do users learn about async failures?" — most candidates have no answer; you do.
**Drill:** find what clears a source's error log and argue whether all-clear-on-any-success is right.

## Pattern 13: Periodic reapers for eventual cleanliness

**Problem:** crash residue (stubs, expired checkouts, trash) accumulates.
**Shape:** beat-scheduled tasks that scan-and-fix, declared next to their queue.
**Real example:** stub deletion + retention checks ([documents/queues.py L50–L69](../../../mayan/apps/documents/queues.py#L50-L69)); checkout expiry via `check_in_expired_check_outs` (checkouts tasks).
**Failure modes:** reaper interval = staleness window; reapers hide root causes (metrics on reap counts!).
**Interview angle:** at-least-once + reapers is the honest alternative to distributed transactions — say exactly that.
**Drill:** what metric on `task_document_stubs_delete` would have caught a broken upload pipeline within a day?

## Pattern 14: Size-bounded artifact cache with locks

**Problem:** rendered page images are expensive, big, and regenerable.
**Shape:** cache/partition/file rows mirroring storage objects; LRU-ish prune under distributed lock before each create.
**Real example:** [CachePartition.create_file, file_caching/models.py L225–L272](../../../mayan/apps/file_caching/models.py#L225-L272); prune loop [L105–L152](../../../mayan/apps/file_caching/models.py#L105-L152).
**Failure modes:** prune ordering by `hits` never decays (old-hot files immortal); DB rows vs storage objects can desync; lock contention on hot partitions.
**Interview angle:** cache stampede/eviction design with a real, non-Redis example.
**Drill:** simulate the prune loop on paper with 3 files and a full cache; find the case where `file_index` resets to 0 and why that's needed ([L119–L131](../../../mayan/apps/file_caching/models.py#L114-L152)).

## Pattern 15: Template-driven extensibility (user-space logic)

**Problem:** admins need per-deployment logic (workflow conditions, index paths, label formats) without code deploys.
**Shape:** Django template snippets stored in DB fields, evaluated with a context.
**Real example:** workflow transition conditions ([workflow_instance_models.py L183–L188](../../../mayan/apps/document_states/models/workflow_instance_models.py#L171-L195)); version label templates ([document_version_models.py L231–L249](../../../mayan/apps/documents/models/document_version_models.py#L231-L249)).
**Failure modes:** templates are code — evaluate the sandboxing story before exposing to non-admins; silent empty-string failures.
**Interview angle:** "how much power do you give configuration?" — a genuinely senior discussion.
**Drill:** find what a malicious admin could reach from a workflow condition template context ([get_context, workflow_instance_models.py L120–L130](../../../mayan/apps/document_states/models/workflow_instance_models.py#L120-L130)).

## Pattern 16: Test-pair convention as an authz contract

**Problem:** every endpoint needs proof that denial works, forever.
**Shape:** for each view, `test_X_no_permission` (expects 404 + no events) and `test_X_with_access` (expects 2xx + exact events).
**Real example:** [test_document_api.py L67–L112](../../../mayan/apps/documents/tests/test_document_api.py#L67-L112); event assertions pin the audit contract too.
**Why it works:** authz regressions fail loudly; the suite doubles as documentation of intended policy (see the change-type asymmetry it *enshrines*, [03/03 §edge cases](03-validation-auth-and-permissions.md)).
**Failure modes:** the pair proves the policy as-written, including wrong policy; combinatorial growth.
**Interview angle:** "how do you test authorization?" — describe the pair + event assertions.
**Drill:** write the missing third member for change-type: `test_..._target_type_no_permission`, and predict its current result (200 — the asymmetry).

---

**Catalog drill:** pick any two cards and find one *more* instance of each pattern elsewhere in the repo (candidates: metadata validators for Pattern 9; favorite documents for Pattern 6's proxies). *Strong:* for one instance, identify where the implementation diverges from the canonical example and whether the divergence is drift or adaptation.
