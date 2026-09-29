# Boundaries and layers

HTTP owns protocol/status/representation; serializers/forms own input shape; permissions own access decisions; domain models/managers own invariants; storage owns bytes; tasks own deferred orchestration; events own audit/integration notifications. The upload view stages and queues but delegates creation to `document.file_new` at [documents/tasks.py](../../mayan/apps/documents/tasks.py#L82-L89).

A boundary leak is not merely “logic in the wrong file”; it is one layer depending on another layer's unstable details. Possible risk: tasks catch broad exceptions and encode cleanup policy at [documents/tasks.py](../../mayan/apps/documents/tasks.py#L90-L124), coupling orchestration to staging lifecycle. Investigate with failure tests before refactoring.

Drill: assign ownership for validation, permission, checksum, retry, audit, and cleanup. Strong: name the contract between each pair.
