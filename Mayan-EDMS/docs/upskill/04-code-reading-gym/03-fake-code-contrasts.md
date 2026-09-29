# Fake Code Contrasts

## Coupling UI shape to DB shape

```python
# Illustrative fake code: not from this repo.
document.label = request.data["file"]["name"]
```

Better shape:

```python
# Illustrative fake code: adapt to the repo.
document_file = document.file_new(file_object=file_obj, _user=request.user)
```

Why: real code centralizes file lifecycle in [`document_models.py`](../../mayan/apps/documents/models/document_models.py#L189-L251).

## Missing permission filter

```python
# Illustrative fake code: not from this repo.
document_type = DocumentType.objects.get(pk=document_type_id)
```

Better shape:

```python
# Illustrative fake code: adapt to the repo.
queryset = AccessControlList.objects.restrict_queryset(...)
document_type = get_object_or_404(queryset=queryset, pk=document_type_id)
```

Real pattern: [`document_api_views.py`](../../mayan/apps/documents/api_views/document_api_views.py#L119-L129)

## N+1 style repeated async work

```python
# Illustrative fake code: not from this repo.
for file in document.files.all():
    task_process_document_file.delay(file.pk)
```

Better shape:

- understand whether only `file_latest` should be processed, as the repo often does in [`file_metadata/methods.py`](../../mayan/apps/file_metadata/methods.py#L7-L12)

## Stale state after partial DOM replacement

```js
// Illustrative fake code: not from this repo.
$('.my-link').click(handler);
```

Better shape:

```js
// Illustrative fake code: adapt to the repo.
$('body').on('click', 'a', function (event) { ... });
```

Real pattern: [`partial_navigation.js`](../../mayan/apps/appearance/static/appearance/js/partial_navigation.js#L273-L280)

## Swallowing errors

```python
# Illustrative fake code: not from this repo.
try:
    do_ocr()
except Exception:
    pass
```

Better shape:

- retry selected exceptions
- persist error visibility

Real pattern: [`ocr/tasks.py`](../../mayan/apps/ocr/tasks.py#L80-L89) and [`ocr/tasks.py`](../../mayan/apps/ocr/tasks.py#L119-L125)

## Overusing ambient request data deep in domain logic

```python
# Illustrative fake code: not from this repo.
def create_document(request, file_object):
    return Document.objects.create(label=request.POST["label"])
```

Better shape:

- pass the narrow values you actually need
- preserve audit context explicitly with `_user`

Real pattern: [`document_type_models.py`](../../mayan/apps/documents/models/document_type_models.py#L138-L176)

## Casual public-contract change

```python
# Illustrative fake code: not from this repo.
url(regex=r'^documents/upload/$', view=NewUploadView.as_view())
```

Better shape:

- preserve route shape unless you are intentionally versioning a contract

Real contract surface: [`documents/urls.py`](../../mayan/apps/documents/urls.py#L528-L533)

## Side effects before durable state

```python
# Illustrative fake code: not from this repo.
send_event("document_created")
document.save()
```

Better shape:

- persist first, then emit the event from the save lifecycle or just after it

Real pattern: [`document_models.py`](../../mayan/apps/documents/models/document_models.py#L292-L317)

## Ignoring archive expansion blast radius

```python
# Illustrative fake code: not from this repo.
for member in zip_file.members():
    process(member)
```

Better shape:

- call out recursion and cost explicitly
- think about quotas and nested archive behavior

Real examples:
- [`document_models.py`](../../mayan/apps/documents/models/document_models.py#L205-L229)
- [`sources/models.py`](../../mayan/apps/sources/models.py#L97-L124)
