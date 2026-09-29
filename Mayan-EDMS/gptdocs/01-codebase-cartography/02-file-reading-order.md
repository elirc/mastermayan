# File reading order

| Order | File | Why / look for |
| --- | --- | --- |
| 1–4 | `README.rst`, `setup.py`, `mayan/settings/base.py`, `mayan/celery.py` | product, dependencies, composition, workers |
| 5–9 | document file API view, serializer, task, model, API test | one complete vertical slice |
| 10–13 | REST permission, ACL manager/model, document permissions | security boundary |
| 14–17 | source API/serializer/model/task | plugin-like ingestion |
| 18–21 | OCR API/model/manager/task | async fan-out and persistence |
| 22–25 | workflow model/task/API/test | state machine |
| 26–29 | storage classes/models/settings/tests | blob ownership |
| 30–33 | events classes/models/tests; logging | audit/observability |

Junior path: 1–17; ignore app registration machinery on first pass. Mid path: 1–29 and compare two vertical slices. Senior path: all, plus migrations, CI, deployment, and cross-app hooks. Pause before each implementation and predict input, output, dependency, side effect, and failure. Verify predictions in tests.
