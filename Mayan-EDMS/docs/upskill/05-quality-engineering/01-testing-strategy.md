# Testing Strategy

## Test layers visible in this repo

| Layer | Examples | Best use |
| --- | --- | --- |
| Model/business tests | `documents/tests/test_*.py`, `checkouts/tests/` | invariants, lifecycle behavior |
| API tests | `*_api.py` files | permission, status, payload contract |
| View tests | `*_views.py` files | UI permissions and flow control |
| Migration tests | `test_migrations.py`, CI migration target | upgrade safety |
| Upgrade tests | [`.gitlab-ci.yml`](../../.gitlab-ci.yml#L179-L248) | previous-release compatibility |

## What not to test

- framework internals already owned by Django/DRF
- exhaustive translation behavior
- every trivial getter

## Flake prevention themes

- isolate by app-local fixtures
- avoid time-sensitive assumptions unless explicitly testing expiration logic
- prefer ID-based reloads for async-style behavior
