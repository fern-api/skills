# CI-only failures

| Symptom | Cause → fix |
| --- | --- |
| `Please run fern login` / `401` / `Unauthorized` | `FERN_TOKEN` missing from the step's `env`, empty on fork/Dependabot runs, or revoked. Regenerate with `fern token`; guard forked PRs. |
| `No fern directory found` / `fern.config.json not found` | Wrong working directory. Set `working-directory:` or `cd` to the folder that contains `fern/`. |
| `Cannot connect to the Docker daemon` | `--local` on a runner without Docker. Use remote generation or a Docker-capable runner/service. |
| `Unable to find image fernapi/...:x.y.z` / pull rate limit | Version typo, or Docker Hub anonymous pull limits on busy runners. Pin a valid version; authenticate to Docker Hub or mirror images and set `image.registry`. |
| `fetch failed` installing/upgrading the CLI | Runner can't reach registry.npmjs.org: proxy (`HTTPS_PROXY`), custom registry (`npm config set registry`), corporate TLS (`NODE_EXTRA_CA_CERTS`). Pin `fern.config.json` to a version already cached if needed. |
| Prompt hangs then job times out | Interactive confirmation outside GitHub Actions (preview `--id` overwrite, release confirm). Add `--no-prompt` / `--force`. |
| Preview URL empty in output parsing | Output format is `Published docs to <url>`; make sure you capture stderr (`2>&1`) and the step didn't fail before printing. |
| `repository not found` / cannot push SDK | Fern GitHub App not installed on the SDK repo, or `github.repository` typo. Install at https://github.com/apps/fern-api. |
| `refusing to allow ... workflows permission` | Token can't write `.github/workflows`. Set `github.workflows: false` or use a token with `workflow` scope. |
| Sync action can't open PR | Token lacks Contents/Pull requests write, or repo setting blocks Actions-created PRs. Use a PAT/App token. |
| `Unrecognized key` in `generators.yml` on CI but not locally | Different CLI version: local has `*`/newer pin. Commit a concrete `fern.config.json` version; run `fern check` with the same pin. |
| Generated SDK PR fails its own CI | Problem is in the SDK repo (tests, formatting, missing registry secret). Fix there or via generator `config`; don't patch generated files by hand unless `.fernignore`d. |
| Different output on each run | Unpinned generator version or unpinned `fern.config.json`. Pin both. |
| Release created but package not on registry | SDK repo publish job failed or wasn't triggered (tag format, missing secret). Check the SDK repo's Actions tab, not the config repo. |

Debugging aids: `--log-level debug`, `fern check --warnings`, run the identical command locally with `FERN_TOKEN` exported, and compare the CLI version printed at the start of the log with `fern.config.json`.
