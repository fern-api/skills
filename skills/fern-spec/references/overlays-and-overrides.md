# Overlays vs overrides vs editing the spec

| Approach | When | Mechanism |
| --- | --- | --- |
| Edit the spec | You own the file in `fern/openapi/` and nothing regenerates it | Direct edit |
| **Overlay** (preferred) | Spec is downloaded (`origin`), generated from code, or shared with other tools | [OpenAPI Overlay 1.0.0](https://spec.openapis.org/overlay/v1.0.0) file: JSONPath `target` + `update`/`remove` actions. Standard, works with non-Fern tooling. |
| Overrides | Existing Fern projects, or when you want a YAML mirror of the spec that is deep-merged | Fern-specific `overrides.yml` merged by key path |

Reference in `generators.yml`:

```yaml
api:
  specs:
    - openapi: ./openapi/openapi.json
      overlays: ./overlays/sdk.yml          # string or list, applied in order
      # overrides: ./overrides.yml          # string or list, applied in order (overrides run first, then overlays)
```

Docs: https://buildwithfern.com/learn/api-definitions/openapi/overlays · https://buildwithfern.com/learn/api-definitions/openapi/overrides

## Overlay example

```yaml
overlay: 1.0.0
info:
  title: Acme SDK customisations
  version: 1.0.0
actions:
  - target: $.paths['/plants'].get
    update:
      x-fern-sdk-group-name: plants
      x-fern-sdk-method-name: list
      x-fern-pagination:
        cursor: $request.cursor
        next_cursor: $response.next_cursor
        results: $response.data
  - target: $.paths['/plants/{plantId}'].get.parameters[?(@.name == 'plantId')]
    update:
      x-fern-parameter-name: id
  - target: $.paths['/internal/metrics']
    remove: true
  - target: $.components.schemas.plant_record
    update:
      x-fern-type-name: Plant
  - target: $                               # document level
    update:
      x-fern-global-headers:
        - header: X-Tenant-Id
          name: tenantId
          optional: true
```

Rules:

- `update` deep-merges objects; arrays are appended, not replaced. To replace an array element, `remove` it then add.
- Fern requires parentheses in JSONPath filters: `[?(@.name == 'plantId')]`, not `[?@.name == 'plantId']`.
- Escape paths with `{}` in bracket notation: `$.paths['/users/{id}']`.
- A `target` that matches nothing is a silent no-op; verify with `fern check --warnings` and by inspecting generated output.

## Overrides example

Same result in Fern's overrides format (mirrors the spec structure, deep-merged):

```yaml
# overrides.yml
x-fern-global-headers:
  - header: X-Tenant-Id
    name: tenantId
    optional: true
paths:
  /plants:
    get:
      x-fern-sdk-group-name: plants
      x-fern-sdk-method-name: list
  /plants/{plantId}:
    get:
      parameters:
        - name: plantId        # match by name
          x-fern-parameter-name: id
  /internal/metrics:
    get:
      x-fern-ignore: true
components:
  schemas:
    plant_record:
      x-fern-type-name: Plant
```

`fern write-overrides` scaffolds an `overrides.yml` listing every operation with its current group/method names, which is the quickest way to rename many methods at once. Overrides add or replace values; to hide something use `x-fern-ignore: true` rather than trying to delete it. The overrides file can be YAML even when the spec is JSON.

## Choosing

1. Downloaded or generated spec → overlay. Keep all Fern extensions there so `fern api update` never clobbers them.
2. Already have `overrides.yml` → keep using it; mixing both is allowed (overrides are applied first, then overlays).
3. Hand-maintained spec in `fern/openapi/` → put extensions directly in the spec; a second file adds indirection for no benefit.
4. Need the same customisations across several Fern projects → overlay (portable, standard).
