# Running `fern generate` and fixing failures

Install: `npm install -g fern-api` (pins to `fern.config.json` at runtime: the CLI re-execs the configured version). Run from the repo root or the `fern/` folder. Add `--log-level debug` to any command when debugging.

## Commands

```bash
fern check [--warnings]                 # validate specs + generators.yml; run before every generate
fern generate                           # default-group, remote generation (needs fern login / FERN_TOKEN)
fern generate --group <g> [--api <a>]   # a specific group / API
fern generate --group <g> --version 1.2.3   # set package version + release tag
fern generate --group <g> --preview     # no publish; writes to fern/.preview/<generator-name>/
fern generate --group <g> --local       # run generator Docker images locally (no login)
fern generate --group <g> --local --preview --force   # fully offline-ish dry run, no prompts
fern generate --group <g> --local --package [--package-mode host|docker]   # artifacts in fern-dist/
fern generate --group <g> --local --package-only   # package, then delete everything except fern-dist/ (and .fernignore'd paths)
fern generate --no-replay --local       # skip Replay for this run
fern generate --fernignore ./custom.fernignore --local
fern generator upgrade [--include-major]   # update generator versions in generators.yml
fern upgrade [--version X] [--yes]      # update CLI version in fern.config.json (+ migrations)
fern downgrade                          # revert the last CLI upgrade
fern login [--device-code]              # browser / device auth; CI uses FERN_TOKEN instead
fern token                              # print a token for CI (store as FERN_TOKEN)
fern api update [--api <a>]             # refresh specs from `origin`
fern add <generator-name> --group <g>   # scaffold a generator entry
fern init --openapi <path|url>          # create fern/ for an existing spec
fern write-overrides                    # scaffold an overrides.yml for the spec
fern write-definition                   # dump the Fern Definition Fern derived from your spec (debugging)
```

Remote vs local: remote generation (default) runs on Fern's infrastructure and handles GitHub pushes and registry publishing. `--local` pulls the generator image (`fernapi/<generator>:<version>`) and runs it with Docker; it can write to `local-file-system` and push to GitHub but tokens must be in the local environment. `--preview` works with both.

## Reading `generators.yml` validation errors

`fern check` reports the YAML path. Typical causes:

- `Unrecognized key` inside `config` → key doesn't exist for that generator version. Confirm on the language configuration page; bump `version` if it was added later.
- `output.location` and fields mismatch → each location has its own required fields (see publishing.md).
- `default-group` names a group that doesn't exist (or an alias with a typo).
- Spec-level errors (`Unexpected property`, `$ref not found`) are spec problems: use `fern-openapi`.

## Common generation failures

| Symptom | Cause / fix |
| --- | --- |
| `Please run fern login` / 401 in CI | No `FERN_TOKEN` in env. Create with `fern token`, store as a secret. Or use `--local`. |
| `Cannot connect to the Docker daemon` with `--local` | Docker isn't running / not available on the runner. Start Docker, or use remote generation. |
| `Unable to find image fernapi/...:x.y.z` | Version doesn't exist; check the changelog. Private registry → set `image.registry`. |
| Push to GitHub fails / "repository not found" | Fern GitHub App not installed on the SDK repo, or `github.repository` typo. |
| `workflows` permission error on push | Add `github.workflows: false` or grant the token `workflows` scope. |
| Version rejected | Use `X.Y.Z` or `X.Y.Z-rcN`. |
| Generated PR overwrote hand-written code | File wasn't in `.fernignore` (whole-file) and repo isn't in `pull-request` mode (Replay). See custom-code.md. |
| New endpoint missing from SDK | Endpoint filtered by `audiences`, `settings.filter`, or `x-fern-ignore`; or spec `origin` not refreshed (`fern api update`). |
| Method named `postUsers`/odd request names | Missing `operationId`; use `x-fern-sdk-method-name` / `x-fern-request-name` (`fern-openapi`). |
| Errors are generic (`ApiError`) not typed | Spec lacks response schemas for error codes; add them or `settings.error-responses`. |
| `--package` produced nothing | Only `local-file-system` outputs are packaged; GitHub/registry outputs are skipped silently. |
| TypeScript `--local` output has only `src/` | Expected for `local-file-system` without `--package`; use `--package` for an installable `.tgz`. |
| Java generation slow / formatting failures | Generator runs spotless; set `FERN_JAVA_SKIP_FORMATTING=true` in the environment to skip. |
| `fern upgrade` → `fetch failed` | The CLI can't reach the npm registry: check `HTTPS_PROXY`, `npm config get registry`, corporate TLS (`NODE_EXTRA_CA_CERTS`). Pin a version manually in `fern.config.json` as a stopgap. |
| Output differs between remote and `--local` | Different generator version pulled; make sure `version` is pinned and the local image isn't stale (`docker pull`). |

## Upgrading

1. `fern upgrade` (CLI) then `fern check`. Read `https://buildwithfern.com/learn/cli-api-reference/cli-reference/cli-changelog` for breaking changes.
2. `fern generator upgrade` (minor/patch) or `--include-major`. Majors may rename `config` keys; the command runs migrations, but review the diff.
3. `fern generate --group <g> --preview` and diff against the current SDK before releasing. Breaking-change notes are in each generator's changelog page.
