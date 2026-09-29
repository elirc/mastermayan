# Command Cheatsheet

| Command | Status | Source |
| --- | --- | --- |
| `python --version` | Verified | local run |
| `pip --version` | Verified | local run |
| `python manage.py check --settings=mayan.settings.testing.development` | Verified failed | local run, missing Django |
| `python manage.py runserver --settings=mayan.settings.development` | Inferred | [`Makefile`](../../Makefile#L351-L353) |
| `python manage.py test --mayan-apps --settings=mayan.settings.testing.development --skip-migrations` | Inferred | [`Makefile`](../../Makefile#L10-L20) |
| `python manage.py test --mayan-apps --settings=mayan.settings.testing.development --no-exclude --tag=migration_test` | Inferred | [`Makefile`](../../Makefile#L27-L29) |
| `cd docs && make html` | Inferred | [`Makefile`](../../Makefile#L130-L133) |
| `make staging-start` | Inferred | [`Makefile`](../../Makefile#L418-L421) |
| `make staging-frontend` | Inferred | [`Makefile`](../../Makefile#L424-L426) |
| `make staging-worker` | Inferred | [`Makefile`](../../Makefile#L428-L431) |
