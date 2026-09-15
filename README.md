# Adobe API Mesh CI/CD Template

> ⚠️ **Important:** Starting from version 2.x, a custom container is required. The Dockerfile can be found in `.github/workflows/images/aio-mesh-runner/Dockerfile`. If you prefer to use a standard GitHub runner without a custom container, please use version 1.x of this template.

This repository is a lightweight starting point for teams that want a repeatable GitHub Actions pipeline for provisioning and updating Adobe API Mesh configurations. Fork it, drop in your mesh definition files, wire up the required Adobe Developer Console credentials, and you will have a push-button deployment path for staging and production meshes.

---

## What You Get

- **Workflow automation** – `.github/workflows/deploy.yaml` handles checkout, authentication via the Adobe I/O CLI, mesh creation (or update), and post-deploy validation.
- **Environment awareness** – pushes to `staging` or `production` automatically pull the corresponding credential set and workspace identifiers.
- **Manual control** – trigger the workflow with `workflow_dispatch` whenever you need an ad-hoc deployment.
- **Extensibility** – add new environments, steps, linters, or notifications by extending the single workflow file.

### Key Capabilities

- **Dynamic `.env` injection** – GitHub repository variables (`ENV_STAGE`, `ENV_PROD`) can store full `.env` payloads, and the workflow materializes them into a runtime `.env` before invoking the API Mesh CLI.
- **Secret materialization** – branch-specific secrets can include encrypted mesh credentials (`MESH_SECRETS_*`) that the workflow writes to `secrets.yaml` on the fly and passes through `--secrets` so sensitive resolvers stay out of Git history.
- **Auto-flag builder** – the workflow inspects the repo for `.env` and `secrets.yaml` and automatically appends the correct `aio api-mesh:*` flags, helping you wire runtime configuration consistently.
- **Provisioning watchdog** – mesh deployments poll `aio api-mesh:status` for up to 10 minutes with friendly logging, failing early if provisioning stalls or ends unexpectedly.
- **Reusable testing workflow** – `.github/workflows/tests.yaml` defines a Newman-based regression suite that runs after deployment (via the `tests` job in `deploy.yaml`) and selects the right Postman environment per branch.

Repository layout (expected):

```
.
├── .github/workflows/deploy.yaml       # Main CI/CD pipeline
├── .github/workflows/tests.yaml        # Reusable Newman regression tests
├── mesh.json                          # API Mesh definition (commit your own)
├── .env                               # Optional runtime variables used by mesh.json
├── tests/
│   └── newman/
│       ├── collection.json            # Postman collection export
│       ├── environment_staging.json   # Environment variables for staging
│       └── environment_production.json # Environment variables for production
└── README.md                          # This documentation
```

> Tip: keep secrets out of `mesh.json` and `.env`. Use GitHub Encrypted Secrets wherever possible.

---

## Prerequisites

1. **Adobe Developer Console access** with permissions to the target organization, project, and workspaces that host your meshes.
2. **Mesh definition files** (`mesh.json` plus any schema/resolver files) committed to the repository.
3. **GitHub repository admin rights** to configure Actions secrets and branch protection rules.
4. **Node.js 20.x** compatibility for any local development or custom steps (the workflow inherits Node 20 from the runner image).
5. **Adobe I/O CLI knowledge** (`aio`) in case you want to run the same commands locally for troubleshooting.

---

## Required GitHub Secrets

Configure the following secrets under **Settings → Secrets and variables → Actions** in your fork. Stage/prod secrets allow the workflow to decide which credentials to use based on the branch being deployed.

| Secret Name | When Used | Description |
| --- | --- | --- |
| `CLIENTID_STAGE`, `CLIENTSECRET_STAGE`, `TECHNICALACCID_STAGE`, `TECHNICALACCEMAIL_STAGE`, `WORKSPACEID_STAGE` | Pushes to `staging` | OAuth client + technical account bound to your staging workspace. Workspace project > workspace > id from workspace.json |
| `CLIENTID_PROD`, `CLIENTSECRET_PROD`, `TECHNICALACCID_PROD`, `TECHNICALACCEMAIL_PROD`, `WORKSPACEID_PROD` | Pushes to `production` | Production equivalents of the above credentials. Workspace: project > workspace > id from workspace.json |
| `IMSORGID` | Both | ims_org_id from workspace.json |
| `ORGID` | Both | project > org > id from workspace.json |
| `PROJECTID` | Both | project > id from workspace.json |

Add any extra secrets referenced by your mesh (for custom resolvers, HTTP headers, etc.) and load them via environment variables or additional steps in the workflow.

### Recommended GitHub Variables

| Variable Name | When Used | Description |
| --- | --- | --- |
| `ENV_STAGE` | Pushes to `staging` | Full contents of the `.env` file you want the staging deployment to consume (multi-line values supported). |
| `ENV_PROD` | Pushes to `production` | Production `.env` payload, typically mirroring secure resolver configuration for production meshes. |

If these variables are present, the workflow writes them to `.env` before running `aio api-mesh:*`. If they are empty, the pipeline falls back to any `.env` file committed in the repository or skips the flag entirely.

#### How Environment Variable Injection Works

The workflow uses an internal environment variable `AIO_ENV_FILE` to handle the `.env` materialization:

1. **Branch detection** — The workflow identifies the current branch (`staging` or `production`).
2. **Variable mapping** — If `ENV_STAGE` or `ENV_PROD` is set, its contents are stored in the `AIO_ENV_FILE` workflow environment variable.
3. **File materialization** — If `AIO_ENV_FILE` is not empty, the workflow writes its contents to `.env` in the repository root with restricted permissions (`chmod 600`).
4. **Flag computation** — The "Compute Mesh Credential Flags" step checks for the presence of `.env` and automatically appends `--env .env` to the mesh CLI commands.

This approach allows you to:
- Keep sensitive environment values out of Git by storing them as GitHub Variables.
- Override a committed `.env` file with branch-specific values.
- Support multi-line `.env` payloads (GitHub Variables preserve newlines).

**Example `ENV_STAGE` variable content:**
```
RESOLVER_API_KEY=sk-staging-xxx
BACKEND_URL=https://api.staging.example.com
DEBUG=true
```

---

### Runner image requirements

The deploy workflow runs inside a lightweight Docker image that ships Node.js 20,
the Adobe AIO CLI **and** the API Mesh plugin, so the job installs nothing at
runtime.

Dockerfile used by the workflow (`.github/workflows/images/aio-mesh-runner/Dockerfile`):

```
FROM node:20-bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

ENV XDG_DATA_HOME=/usr/local/share

RUN npm install -g @adobe/aio-cli \
    && aio plugins:install @adobe/aio-cli-plugin-api-mesh \
    && mkdir -p /tmp/homecheck \
    && HOME=/tmp/homecheck aio api-mesh --help > /dev/null \
    && npm cache clean --force \
    && rm -rf /root/.npm /tmp/*

WORKDIR /workspace
```

#### Why `XDG_DATA_HOME` is pinned

oclif — the framework behind `aio` — resolves installed plugins from
`$XDG_DATA_HOME`, falling back to `$HOME/.local/share`. In GitHub container jobs
the runner sets `HOME=/github/home`, which is not the `HOME` in effect during
`docker build` (`/root`). A plugin baked in without pinning the data directory
therefore ends up wired to the build-time home and is invisible at run time —
`aio api-mesh:*` fails with `api-mesh is not a aio command`. Earlier versions of
this template worked around that by reinstalling the plugin on every run.

Setting `XDG_DATA_HOME` to an absolute path makes the plugin resolve under any
`$HOME`, so it can be baked in. `@adobe/aio-lib-core-config` reads the same
variable, so `aio config` and the auth context stay consistent too.

The `HOME=/tmp/homecheck aio api-mesh --help` line is a build-time assertion: it
reproduces the exact failure mode above, so the image build fails rather than
shipping a runner whose plugin only works under one `HOME`.

#### Building the image

Pushes to `main` that touch the Dockerfile trigger
`.github/workflows/build-image.yaml`, which builds `linux/amd64` and pushes
`ghcr.io/<owner>/aio-mesh-runner:latest` plus a `:<commit-sha>` tag for
rollbacks. It can also be run by hand from the **Actions** tab.

To build and push manually:

```
docker buildx build \
  --platform linux/amd64 \
  -t ghcr.io/<org-or-user>/aio-mesh-runner:latest \
  --push \
  .github/workflows/images/aio-mesh-runner
```

To verify a built image behaves the way a GitHub container job will run it:

```
docker run --rm -e HOME=/github/home ghcr.io/<org-or-user>/aio-mesh-runner:latest \
  sh -c 'mkdir -p /github/home && aio plugins && aio api-mesh --help'
```

`aio plugins` must list `@adobe/aio-cli-plugin-api-mesh`.

Reference the image in `deploy.yaml`:
```
jobs:
  deploy:
    runs-on: ${{ matrix.os }}
    container:
      image: ghcr.io/<org-or-user>/aio-mesh-runner:latest
    strategy:
      matrix:
        os: [ubuntu-latest]
    # ...

```

**If you do not have such a runner image, use the 1.x version of this template, where:**

- The job runs directly on `ubuntu-latest` (no `container`: stanza).
- `aio` and the API Mesh plugin are installed inside the workflow steps (for example using `adobe/aio-cli-setup-action` plus an explicit `aio plugins:install @adobe/aio-cli-plugin-api-mesh`).

In short:

- 2.x template – requires a prebuilt runner image with `aio` and the API Mesh plugin; no CLI installation steps in the workflow.
- 1.x template – no custom image required; CLI and plugin are installed at runtime in the job, which is slower but does not depend on Docker image management.

### Developer Terms of Service

`aio console:project:select` and `aio console:workspace:select` prompt for the
Adobe Developer Terms of Service if the organization has not accepted them. There
is no non-interactive flag for that prompt, and with no TTY the job blocks and
exits with code 130. Both steps therefore answer it on stdin:

```
- name: Select project
  run: yes | aio console:project:select ${{ secrets.PROJECTID }}
```

Acceptance is recorded per organization on Adobe's side, so this fires once. If
you would rather not accept the terms from CI, accept them once in the Adobe
Developer Console and drop the `yes |` prefixes.

## Quick Start

1. **Fork this repo** or use it as a template inside your organization.
2. **Add your mesh files**:
	- Place the primary mesh definition in `mesh.json`.
	- Commit any supporting schemas/resolvers alongside it.
	- Provide runtime configuration via Git-tracked `.env` files **or** populate the `ENV_STAGE` / `ENV_PROD` GitHub variables with the exact `.env` content you want injected per environment.
3. **Populate GitHub Secrets** with the values listed above.
4. **Adopt the branch convention**:
	- Push or merge to `staging` for deploying to staging Adobe workspaces.
	- Push or merge to `production` for production deployments.
5. **Run the pipeline**:
	- Commit changes and push to the target branch.
	- Or open the **Actions** tab, select **Deploy Mesh**, and click **Run workflow** (specify the target branch).

Once the workflow succeeds, your Adobe API Mesh instance will be created (if missing) or updated using the latest `mesh.json` contents, then described and queried for status to confirm a healthy deployment.

---

## Workflow Walkthrough (`deploy.yaml`)

1. **Checkout** – verifies the baked-in `aio` CLI and API Mesh plugin, then pulls repository code. Node 20 comes from the runner image; nothing is installed at runtime.
2. **Branch-aware env mapping** – resolves GitHub secrets to runtime variables (`TARGET_ENV`, `CLIENTID`, etc.). Only `staging` and `production` are allowed to prevent accidental deployments from other branches.
3. **Secret validation** – fails fast if any required value is missing.
4. **Authentication & targeting** – performs `oauth_sts`, selects the right org/project/workspace, and prints the CLI config for traceability.
5. **Mesh lifecycle** – runs `aio api-mesh:get`; if no mesh exists it calls `api-mesh:create`, otherwise `api-mesh:update`, then waits briefly, describes the mesh, and fetches status.

Extend or reorder steps as needed (e.g., run linting/tests before deployment, send Slack notifications after success, etc.).

---

## Newman Testing Configuration

The template includes a reusable testing workflow (`.github/workflows/tests.yaml`) that runs Newman-based regression tests after each successful deployment. Tests are **optional** — the workflow gracefully skips execution if the required files are not present.

### Required Folder Structure

```
tests/
└── newman/
    ├── collection.json              # Postman collection export (required)
    ├── environment_staging.json     # Environment variables for staging branch
    └── environment_production.json  # Environment variables for production branch
```

### File Details

| File | Description |
| --- | --- |
| `collection.json` | Exported Postman collection containing your API tests. Export from Postman using **Collection → Export → Collection v2.1**. |
| `environment_staging.json` | Postman environment export with variables for staging (e.g., `baseUrl`, API keys). The filename must match `environment_staging.json`. |
| `environment_production.json` | Postman environment export with variables for production. The filename must match `environment_production.json`. |

### How It Works

1. After the `deploy` job completes successfully, the `tests` job is triggered.
2. The workflow checks for `tests/newman/collection.json` — if missing, tests are skipped.
3. The workflow looks for `tests/newman/environment_<branch>.json` based on the current branch — if missing, tests are skipped.
4. If both files exist, Newman runs inside a Docker container (`postman/newman_alpine33`) and executes all requests in the collection against the branch-specific environment.

### Creating Your Test Files

1. **Export your Postman collection:**
   - Open Postman and select your collection.
   - Click the **...** menu → **Export** → **Collection v2.1** → Save as `collection.json`.

2. **Export your Postman environments:**
   - Go to **Environments** in Postman.
   - Select the environment → **...** → **Export** → Save as `environment_staging.json` or `environment_production.json`.

3. **Place files in the correct location:**
   ```bash
   mkdir -p tests/newman
   mv collection.json tests/newman/
   mv environment_staging.json tests/newman/
   mv environment_production.json tests/newman/
   ```

### Example Environment File

```json
{
  "id": "abc123",
  "name": "staging",
  "values": [
    {
      "key": "baseUrl",
      "value": "https://your-mesh-staging.adobeioruntime.net/api/v1/web/mesh",
      "enabled": true
    },
    {
      "key": "apiKey",
      "value": "your-staging-api-key",
      "enabled": true
    }
  ]
}
```

> **Tip:** Avoid committing sensitive values in environment files. Use Postman's variable substitution with placeholder values and override them via GitHub Secrets if needed.

---

## Customizing the Template

- **Additional environments** – duplicate the branch/secrets mapping block and add new branches like `qa` or `dev` with their own credential sets.
- **Runtime changes** – the Node version comes from the runner image, not the workflow; change the `FROM` line in the Dockerfile. Adjust `matrix.os` if you deploy from self-hosted runners.
- **Multiple meshes** – add extra steps to iterate over multiple `mesh.json` files or parameterize the mesh name via `.env` values.
- **Observability** – append steps that push deployment metadata to your logging/monitoring stack.

When editing `deploy.yaml`, keep the secret-validation step up to date so failures happen quickly.

---

## Local Validation (Optional)

If you mirror the GitHub secrets as local environment variables you can run the same commands for smoke tests:

```bash
npm install -g @adobe/aio-cli
aio plugins:install @adobe/aio-cli-plugin-api-mesh
aio console:org:select <ORGID>
aio console:project:select <PROJECTID>
aio console:workspace:select <WORKSPACEID>
aio api-mesh:update -c mesh.json --env .env
```

This makes it easier to debug mesh misconfigurations before committing.

---

## Troubleshooting

- **Unsupported branch error** – ensure you are pushing to `staging` or `production`, or extend the branch mapping block for additional environments.
- **Secret validation failure** – confirm every secret listed earlier is present; GitHub scope is repository-level by default.
- **CLI auth issues** – verify the OAuth client has `oauth_sts` permissions and the scopes defined in the workflow.
- **Mesh update fails** – inspect the workflow logs around `aio api-mesh:update` for schema/validation errors; run the same command locally with `--verbose` for more detail.

---

## Next Steps

- Protect the `production` branch to require PR reviews before deployment.
- Wire Slack/Teams notifications by adding an extra step after `Get Mesh Status`.
- Version your mesh definitions (e.g., tag releases) so you can roll back quickly if needed.

Happy shipping!
