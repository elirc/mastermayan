# Tooling and Build System

## Commands that matter most

- Test orchestration: [`Makefile`](../../../Makefile#L10-L96)
- Docs build: [`Makefile`](../../../Makefile#L130-L139)
- Staging helpers: [`Makefile`](../../../Makefile#L418-L431)
- CI matrix: [`.gitlab-ci.yml`](../../../.gitlab-ci.yml#L249-L346)

## Mental model

- `Makefile` is the human-friendly interface.
- `.gitlab-ci.yml` is the "what production-like validation actually runs" source of truth.
- Docker Compose describes realistic service topology.

## Sharp edges

- `tox.ini` is historically interesting but appears older than the main CI story.
- Local Python commands will fail until dependencies are installed.
- Some commands assume Linux tooling (`apt`, container CLIs).

## Drill

- Compare `test-all`, `test-all-migrations`, and GitLab’s upgrade tests. What different risks do they cover?
