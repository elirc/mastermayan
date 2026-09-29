# Verification log

Running log of what was inspected while authoring this curriculum. Date: 2026-07-11. Repo: Mayan EDMS v4.3.1 (`__init__.py` at repo root: `__version__ = '4.3.1'`, `__django_version__ = '3.2'`). The working copy has **no git history** (`git ls-files` returns nothing inside `Mayan-EDMS/`; the directory sits inside an unrelated user-level git repo).

## Environment

- Host: Windows 11, Python 3.13.3 available locally. Mayan 4.3.1 targets Django 3.2 / older Python (uses `ugettext_lazy`, removed in Django 4); **no attempt was made to install or run the app or its tests locally**. All commands in the curriculum are marked `inferred` unless stated otherwise.
- Docker deployment definitions exist (`docker/docker-compose.yml`) and were read, not run.

## Files read in full

- `mayan/apps/documents/models/document_models.py` (337 lines)
- `mayan/apps/documents/models/document_file_models.py` (536 lines)
- `mayan/apps/documents/models/document_version_models.py` (389 lines)
- `mayan/apps/documents/tasks.py` (333 lines)
- `mayan/apps/acls/managers.py` (383 lines)
- `mayan/apps/file_caching/models.py` (436 lines)
- `mayan/apps/events/classes.py` (493 lines), `events/decorators.py` (33 lines)
- `mayan/apps/checkouts/models.py` (137 lines)
- `mayan/apps/sources/tasks.py` (103 lines)
- `mayan/apps/lock_manager/backends/file_lock.py`, `model_lock.py`
- `mayan/apps/dynamic_search/handlers.py`, `dynamic_search/tasks.py`
- `mayan/apps/document_states/models/workflow_instance_models.py` (311 lines)
- `mayan/apps/rest_api/generics.py`, `rest_api/api_view_mixins.py`
- `mayan/apps/documents/api_views/document_api_views.py`, `serializers/document_serializers.py`
- `mayan/apps/common/apps.py` lines 1–120 (`MayanAppConfig`)
- `mayan/apps/documents/queues.py`, `task_manager/workers.py`, `mayan/celery.py`

## Files read in part / outline (grep of class and def lines plus targeted ranges)

- `mayan/apps/sources/source_backends/mixins.py` (lines 24–121), `watch_folder_backends.py` (lines 1–120)
- `mayan/apps/views/mixins.py` (outline + lines 549–660)
- `mayan/apps/permissions/classes.py` (outline + `check_user_permissions`), `permissions/models.py` (`user_has_this`, lines 193–216)
- `mayan/apps/ocr/tasks.py` (lines 1–80)
- `mayan/apps/duplicates/duplicate_backends.py` (lines 1–40), `duplicates/models.py` + `managers.py` outlines
- `mayan/apps/documents/search.py` (lines 1–80)
- `mayan/apps/documents/views/document_views.py` (lines 1–90)
- `mayan/apps/metadata/models.py` (outline)
- `mayan/apps/converter/classes.py` (outline)
- `mayan/apps/testing/tests/base.py` and `testing/tests/mixins.py` (outlines)
- `Makefile` (lines 1–60), `.gitlab-ci.yml` (lines 1–50), `docker/docker-compose.yml` (lines 1–30), `requirements.txt`, `requirements/base.txt` (lines 1–25), `mayan/settings/base.py` (lines 1–50), `mayan/urls/base.py`, `mayan/rest_api/urls.py`

## Commands run

| Command | Result |
| --- | --- |
| `ls`, `find`-style inventories over `mayan/apps` | 57 Django apps enumerated |
| `find mayan -name "*.py" \| wc -l` | 2,181 Python files |
| `grep -rn "pages_append_all\|task_document_version_page_list_append" mayan/apps` | 6 call sites; used to trace the append-pages flow |
| `grep -rn "modification" mayan/apps/documents/tests/test_document_version_views.py test_document_version_api.py` | no matches — no direct view/API test exercising version modifications was found by this grep |
| `grep -n "page_number" mayan/apps/documents/models/document_file_page_models.py` | confirms `page_number` lives on `DocumentFilePage`, not `DocumentFile` |
| `python --version` | 3.13.3 (too new to run this Django 3.2 codebase; nothing was executed against the app) |

## Uncertainties and hypotheses (kept as "investigate" in the docs)

1. **`pages_append_all` ordering** — `document_file__page_number` is not a field of `DocumentFile`; expected to raise `FieldError` when the queryset is evaluated. Not executed; labeled *investigate: probable defect* wherever cited.
2. **`DocumentFile.delete()` sets `is_stub = False` when the last file is removed** — reads inverted relative to the `is_stub` help text. Labeled *investigate*.
3. **`ignore_results=True` typo** in `documents/tasks.py` `task_document_upload` decorator (valid Celery kwarg is `ignore_result`). Effect (results silently kept) not verified at runtime.
4. **`SortingViewMixin`** passes the raw `?_ordering=` query parameter into `order_by()`. Whether some upstream layer validates it was not found; labeled *possible risk*.
5. **`FileLock._release`** catches `EOFError`, but `json.loads('')` raises `JSONDecodeError`; the module-level `threading.Lock` is not released on that path. Static reading only.
6. **Checkout TOCTOU** — `DocumentCheckout.save()` does a check-then-act (`is_checked_out()` then `save()`); the `OneToOneField` makes the DB reject the loser with `IntegrityError`, not `DocumentAlreadyCheckedOut`. Static reading only.
7. **`task_deindex_instance`** re-fetches the instance from the DB; if the row is already deleted when the worker runs, the `.get()` raises unhandled `DoesNotExist`. Where the deindex signal connects (pre vs post delete) was not fully traced.
8. All behavior claims about Celery chords, retries, and locking semantics come from reading code and Celery 5.2.3 docs knowledge, not from observing the running system.

## Areas not covered

Ran out of depth budget rather than interest: `mirroring/`, `signature_captures/`, `django_gpg/` internals, `redactions/` math, `mayan_statistics/`, most of `appearance/` JS, the Vagrant/Portainer contrib trees, and the Sphinx `docs/` tree. The pre-existing `gptdocs/` directory (an earlier, thinner generated curriculum) was left untouched.
