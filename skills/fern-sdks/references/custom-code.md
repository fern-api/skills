# Keeping custom code across regenerations

Fern owns every file it generates and rewrites them on each run. Two mechanisms protect hand-written code; pick based on granularity.

| | `.fernignore` | Replay |
| --- | --- | --- |
| Granularity | Whole files/paths | Individual lines inside generated files |
| How | List paths in `fern/.fernignore` (gitignore syntax) in the **SDK repo** | 3-way merge of your edits onto the regenerated output |
| Requirements | Any GitHub mode or local output | `github.mode: pull-request`; state in `.fern/replay.lock` in the SDK repo |
| Typical use | `README.md`, `LICENSE`, `.github/workflows/*.yml`, `src/custom/**`, extra test files | Tweaking a generated method, adding an import, adjusting a default |

Both apply to SDKs only, not to docs output.

## `.fernignore`

`.fernignore` lives at the root of the generated SDK repository (Fern creates it on first publish when using GitHub modes). Add one path per line:

```
# .fernignore
README.md
src/custom/
src/Client.custom.ts
tests/integration/
```

Workflow for adding a custom method:

1. Put the new code in its own file (`src/custom/payments.ts`) and list that file in `.fernignore`.
2. Expose it by extending or wrapping the generated client instead of editing the generated client file. Each language page has the idiomatic pattern (`https://buildwithfern.com/learn/sdks/generators/<lang>/custom-code`): TypeScript subclasses the generated client in a custom file and re-exports it; Python subclasses the generated `BaseClient` in a custom module; Java/C# use a wrapper or partial class; Go adds methods in a separate file in the same package.
3. Regenerate with `--preview` and confirm the custom file is still present and compiles.

If you need to change a *generated* file and can't move the change elsewhere, use Replay or fix the source (spec extension or generator `config`) so the generated output is right in the first place. Generator-level fixes are preferred: they work for all languages and don't need per-repo maintenance.

Local generation honours `.fernignore` too: `fern generate --local --fernignore ./path/to/.fernignore`.

## Replay

With `github.mode: pull-request`, Fern records the diff between what it generated and what is on the default branch, and re-applies that diff on the next generation. Conflicts surface in the PR as conflict markers.

```bash
fern replay resolve   # after hand-fixing conflict markers in the SDK repo
fern replay forget    # drop recorded edits for the current API/generator
fern generate --no-replay --local   # one-off local run without Replay
```

Disable globally with `replay: { enabled: false }` in `generators.yml`. Replay is not a substitute for `.fernignore` for files you own entirely.

## Changing the generated surface instead

Many "customisations" are really spec or config changes and should be done there:

| Want | Do |
| --- | --- |
| Rename a method / group | `operationId`, `x-fern-sdk-method-name`, `x-fern-sdk-group-name` in the spec |
| Rename a request wrapper (`ListUsersRequest` vs `UsersListRequest`) | `x-fern-request-name`, or `api.specs[].settings.idiomatic-request-names` |
| Typed exceptions per status code | Spec error responses with schemas, or `api.specs[].settings.error-responses` (`schema`, `apply-to`, `ensure`) |
| Retries | `x-fern-retries` on the operation; generator `config` for defaults where supported |
| Hide endpoints/schemas | `x-fern-ignore`, `x-fern-audiences` + `groups.<g>.audiences`, or `settings.filter` |
| Different client class name / package | Generator `config` (`client_class_name`, `namespaceExport`, `module.path`, …) |
| Extra README content | `readme` block in `generators.yml`, or add `README.md` to `.fernignore` |
| CI tweaks in the SDK repo | `.github/workflows/*.yml` in `.fernignore` (or `github.workflows: false`) |

## Testing the SDK after regeneration

Generated SDK repos contain a test suite (unit tests plus wire tests generated from spec examples). Run the language's standard command (`pnpm test` / `pytest` / `./gradlew test` / `go test ./...` / `dotnet test` / `composer test` / `bundle exec rspec`) inside the preview or PR checkout before releasing, and keep integration tests in `.fernignore`d files.
