# Tooling and build system

`setup.py` is packaging/runtime dependency metadata; Make wraps development, tests, docs, containers, releases, and checks. `tox.ini` is legacy-looking evidence, not the source of truth. Before changing dependencies: identify generation templates (`setup.py.tmpl`), requirement generation targets, CI use, and supported Python matrix.

Drill: follow `make test MODULE=...` without running it and write the expanded command. Then compare CI configuration. Strong: identify which generated files should not be hand-edited and propose a reproducible verification command.

## Interview angle

Expect: “How do you orient in an unfamiliar build?” Answer by naming manifests, wrappers, CI truth, generated artifacts, and one cheap validation.
