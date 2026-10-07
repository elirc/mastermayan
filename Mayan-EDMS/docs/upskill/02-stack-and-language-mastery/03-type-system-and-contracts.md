# Type System and Contracts

This repo is Python-first, so contracts are mostly enforced by runtime validation, ORM schemas, serializer fields, and permission maps rather than a compile-time type checker.

## Contract layers

| Layer | Example | Contract form |
| --- | --- | --- |
| ORM schema | [`document_models.py`](../../../mayan/apps/documents/models/document_models.py#L53-L104) | field types, nullability, indexes |
| Runtime validation | [`checkouts/models.py`](../../../mayan/apps/checkouts/models.py#L68-L72) | `clean()` |
| Serializer/API contract | [`document_api_views.py`](../../../mayan/apps/documents/api_views/document_api_views.py#L113-L135) | serializer-driven request shape |
| Permission contract | [`document_api_views.py`](../../../mayan/apps/documents/api_views/document_api_views.py#L32-L37) | per-method permission map |
| YAML plugin config | [`document_type_models.py`](../../../mayan/apps/documents/models/document_type_models.py#L70-L76) | validated text config |

## What a mid-level engineer should internalize

- Python code without type hints still has contracts.
- The absence of static typing increases the value of tests and narrow interfaces.
- Public contracts are route + serializer + permission + side-effect expectations.

## Drill

- Find one contract that is not obvious from field types alone. Suggested answer surface: `DocumentCheckout.save()` and `Document.file_new()`.
