# Command cheatsheet

No command below was executed during curriculum creation; all are **inferred** from Make targets or Django conventions.

| Goal | Command | Status |
| --- | --- | --- |
| Dev setup | `make setup-dev-environment` | inferred; likely Unix/native deps |
| Server | `make runserver` | inferred |
| Django command | `make manage COMMAND="check"` | inferred; confirm variable name in Makefile |
| Target test | `make test MODULE=documents.tests.test_document_file_api` | inferred |
| All tests | `make test-all` | inferred |
| Migration tests | `make test-all-migrations` | inferred |
| Missing migrations | `make check-missing-migrations` | inferred |
| Coverage | `make coverage-run && make coverage-html` | inferred; run separately on PowerShell |
| Docs | `make docs-html` | inferred |
| Docs live | `make docs-serve` | inferred |
| Dependency safety | `make safety-check` | inferred |
| PostgreSQL services | `make staging-start` | inferred; Docker required |

Before running setup, read the relevant Make recipe. The repository includes historical metadata; confirm supported OS/Python/services with maintainers.
