# Contracts without TypeScript

Python here relies on runtime contracts: serializer fields, Django model fields, method signatures, and tests. `DocumentFileSerializer` distinguishes create-only and read-only fields at [document_file_serializers.py](../../mayan/apps/documents/serializers/document_file_serializers.py#L125-L140). Dynamic action JSON is flexible but less statically discoverable at [sources/serializers.py](../../mayan/apps/sources/serializers.py#L49-L63).

Transfer to TypeScript: serializers correspond to runtime schema validation, not interfaces alone. `unknown` plus parsing is safer than `any`; discriminated unions can model backend-specific actions; generated API types reduce drift only if the runtime schema remains authoritative.

Drill: write a TS discriminated union for two source actions as fake code, then list what still needs runtime validation.

## Interview angle

Practice Q8–Q12 in [runtime deep dive](../08-interview-prep/01-js-ts-node-deep-dive.md).
