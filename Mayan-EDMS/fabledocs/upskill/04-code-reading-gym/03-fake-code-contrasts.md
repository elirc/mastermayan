# Fake-code contrasts

Each pair: a shape a junior would write (bad) vs the shape this repo actually uses (better), with the real anchor. All snippets are labeled fake; the anchors are real.

## Contrast 1: authorization at the call site vs at the queryset

```python
# Illustrative fake code: not from this repo (BAD)
def document_detail(request, pk):
    document = Document.objects.get(pk=pk)          # loads regardless of rights
    if not user_can_view(request.user, document):   # per-view remembering
        raise PermissionDenied                      # 403 leaks existence
    ...
```

Better: never load what the user can't see — filter first, 404 on absence. Real: [restrict_queryset](../../../mayan/apps/acls/managers.py#L268-L294) applied by the base classes ([rest_api/generics.py L120–L133](../../../mayan/apps/rest_api/generics.py#L120-L133)). Failure the bad shape invites: one forgotten `if` = IDOR.

## Contrast 2: bytes in the queue vs IDs in the queue

```python
# Illustrative fake code: not from this repo (BAD)
task_process_upload.delay(file_bytes=uploaded.read(), user=request.user)
```

Broker bloat, pickled users, poison messages you can't inspect. Real: staging row + primitive IDs ([document_serializers.py L80–L88](../../../mayan/apps/documents/serializers/document_serializers.py#L74-L90); [documents/tasks.py L52–L72](../../../mayan/apps/documents/tasks.py#L52-L72)).

## Contrast 3: catch-everything vs retry taxonomy

```python
# Illustrative fake code: not from this repo (BAD)
try:
    process(document_id)
except Exception:
    self.retry()        # retries logic bugs forever; hides the stack trace
```

Real: retry *named* transient errors, surface the rest with context ([dynamic_search/tasks.py L60–L87](../../../mayan/apps/dynamic_search/tasks.py#L41-L89); [documents/tasks.py L37–L45](../../../mayan/apps/documents/tasks.py#L37-L45)).

## Contrast 4: read-modify-write vs F() expressions

```python
# Illustrative fake code: not from this repo (BAD)
cache_file.hits = cache_file.hits + 1   # two workers read 5, both write 6
cache_file.save()
```

Real: `update(hits=F('hits') + 1)` — the DB does the math atomically ([file_caching/models.py L415](../../../mayan/apps/file_caching/models.py#L406-L416)).

## Contrast 5: UI-shape coupled to DB-shape

```python
# Illustrative fake code: not from this repo (BAD)
return JsonResponse({f.name: getattr(doc, f.name) for f in doc._meta.fields})
```

Every schema change becomes an API change; write-only fields leak. Real: explicit serializer field lists with `read_only`/`write_only`/`create_only` intent ([document_serializers.py L43–L62](../../../mayan/apps/documents/serializers/document_serializers.py#L43-L62)).

## Contrast 6: state column UPDATE vs event-sourced state

```python
# Illustrative fake code: not from this repo (NAIVE, not always bad)
instance.state = 'approved'
instance.save()            # who? when? from what state? history gone
```

Real: append a log entry; state is derived from the last one ([workflow_instance_models.py L132–L143](../../../mayan/apps/document_states/models/workflow_instance_models.py#L132-L143)). Honest tradeoff: the naive shape is fine for a boolean; the log shape pays off when audit/compliance/escalation exist ([check_escalation L63–L80](../../../mayan/apps/document_states/models/workflow_instance_models.py#L63-L80)).

## Contrast 7: swallowed errors vs entity error logs

```python
# Illustrative fake code: not from this repo (BAD)
except Exception:
    logger.error('import failed')   # ops sees it; the admin who owns the folder never does
```

Real: write to the *source's* error log so the owner sees it in the UI, clear on success ([sources/tasks.py L39–L53](../../../mayan/apps/sources/tasks.py#L39-L53)).

## Contrast 8: hand-rolled singleton vs registry classes

```python
# Illustrative fake code: not from this repo (BAD)
EVENT_TYPES = {}
def register_event(name, **stuff):   # naked dict, no behavior, no validation
    EVENT_TYPES[name] = stuff
```

Real: classes whose `__init__` registers `self`, with `get()`/`all()` and sorting behavior attached ([events/classes.py L323–L354](../../../mayan/apps/events/classes.py#L323-L354)). Same idea, but instances carry methods (`commit()`), and the ID scheme is namespaced.

## Contrast 9: sleep-based coordination vs locks with timeouts

```python
# Illustrative fake code: not from this repo (BAD)
time.sleep(random.random())   # "should" avoid two workers scanning the folder
process_folder()
```

Real: named lock, skip-if-held, expiry so crashes can't wedge the source forever ([sources/tasks.py L24–L33](../../../mayan/apps/sources/tasks.py#L18-L55)).

## Contrast 10: any-typed kwargs bag vs explicit task signature

```python
# Illustrative fake code: not from this repo (BAD)
def task_do_upload(**kwargs):    # discover the contract at runtime, in prod
    doc = Document.objects.get(pk=kwargs['docid'])
```

Real: named, defaulted parameters that document the queue contract ([documents/tasks.py L52–L55](../../../mayan/apps/documents/tasks.py#L52-L55)). (And the cautionary twin: even named kwargs rot — `ignore_results=` typo at [L140](../../../mayan/apps/documents/tasks.py#L140-L145).)

## Contrast 11: N+1 template conditions vs bulk exclusion (repo's own weak spot)

```python
# Illustrative fake code: not from this repo (this one mirrors REAL repo code)
for entry in queryset:                     # render a template per row,
    if not entry.evaluate_condition(...):  # re-derive queryset each miss
        queryset = queryset.exclude(id=entry.pk)
```

That *is* the shape at [workflow_instance_models.py L184–L188](../../../mayan/apps/document_states/models/workflow_instance_models.py#L171-L195) — included so you practice critiquing production code, not just strawmen. Better shape: collect excluded IDs into a list, one `exclude(id__in=...)` at the end. When you find a "bad shape" in a good codebase, the review question is *does it matter here?* (transition lists are short — probably not; say so).

## Contrast 12: cleanup in the happy path only

```python
# Illustrative fake code: not from this repo (BAD)
process(staged_file)
staged_file.delete()    # exception above = orphan forever
```

Real: deletion in success *and* failure branches, with the failure branch itself guarded ([documents/tasks.py L103–L124](../../../mayan/apps/documents/tasks.py#L103-L124)) — plus a periodic reaper as the backstop ([pattern 13](../03-architecture-and-patterns/05-pattern-catalog.md)).

**Drill:** for contrasts 1, 3, and 12, write the *test* that fails on the bad shape and passes on the good one. *Strong:* your contrast-1 test is exactly the house pair convention ([test_document_api.py L138–L168](../../../mayan/apps/documents/tests/test_document_api.py#L138-L168)).
