# Adobe API Mesh CI/CD Template

This repository is a lightweight starting point for teams that want a repeatable GitHub Actions pipeline for provisioning and updating Adobe API Mesh configurations. Fork it, drop in your mesh definition files, wire up the required Adobe Developer Console credentials, and you will have a push-button deployment path for staging and production meshes.

---

## What You Get

- **Workflow automation** – `.github/workflows/deployMesh.yaml` handles checkout, authentication via the Adobe I/O CLI, mesh creation (or update), and post-deploy validation.
- **Environment awareness** – pushes to `staging` or `production` automatically pull the corresponding credential set and workspace identifiers.
- **Manual control** – trigger the workflow with `workflow_dispatch` whenever you need an ad-hoc deployment.
- **Extensibility** – add new environments, steps, linters, or notifications by extending the single workflow file.

Repository layout (expected):

```
.
├── .github/workflows/deployMesh.yaml   # Main CI/CD pipeline
├── mesh.json                          # API Mesh definition (commit your own)
├── .env                               # Optional runtime variables used by mesh.json
└── README.md                          # This documentation
```

> Tip: keep secrets out of `mesh.json` and `.env`. Use GitHub Encrypted Secrets wherever possible.

---

## Prerequisites

1. **Adobe Developer Console access** with permissions to the target organization, project, and workspaces that host your meshes.
2. **Mesh definition files** (`mesh.json` plus any schema/resolver files) committed to the repository.
3. **GitHub repository admin rights** to configure Actions secrets and branch protection rules.
4. **Node.js 20.x** compatibility for any local development or custom steps (the workflow pins Node 20 via the matrix but can be adjusted).
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

---

## Quick Start

1. **Fork this repo** or use it as a template inside your organization.
2. **Add your mesh files**:
	- Place the primary mesh definition in `mesh.json`.
	- Commit any supporting schemas/resolvers alongside it.
	- (Optional) store non-secret runtime values in `.env` (e.g., `MESH_NAME=my-mesh`). The workflow passes `--env .env` to the CLI so those values are merged during create/update.
3. **Populate GitHub Secrets** with the values listed above.
4. **Adopt the branch convention**:
	- Push or merge to `staging` for deploying to staging Adobe workspaces.
	- Push or merge to `production` for production deployments.
5. **Run the pipeline**:
	- Commit changes and push to the target branch.
	- Or open the **Actions** tab, select **Deploy Mesh**, and click **Run workflow** (specify the target branch).

Once the workflow succeeds, your Adobe API Mesh instance will be created (if missing) or updated using the latest `mesh.json` contents, then described and queried for status to confirm a healthy deployment.

---

## Workflow Walkthrough (`deployMesh.yaml`)

1. **Checkout & Node setup** – pulls repository code and provisions Node 20 on `ubuntu-latest`.
2. **Branch-aware env mapping** – resolves GitHub secrets to runtime variables (`TARGET_ENV`, `CLIENTID`, etc.). Only `staging` and `production` are allowed to prevent accidental deployments from other branches.
3. **Secret validation** – fails fast if any required value is missing.
4. **Adobe I/O CLI bootstrap** – installs `aio` plus the API Mesh plugin (`@adobe/aio-cli-plugin-api-mesh`).
5. **Authentication & targeting** – performs `oauth_sts`, selects the right org/project/workspace, and prints the CLI config for traceability.
6. **Mesh lifecycle** – runs `aio api-mesh:get`; if no mesh exists it calls `api-mesh:create`, otherwise `api-mesh:update`, then waits briefly, describes the mesh, and fetches status.

Extend or reorder steps as needed (e.g., run linting/tests before deployment, send Slack notifications after success, etc.).

---

## Customizing the Template

- **Additional environments** – duplicate the branch/secrets mapping block and add new branches like `qa` or `dev` with their own credential sets.
- **Matrix changes** – adjust `matrix.node-version` or `os` if you need different runtimes.
- **Multiple meshes** – add extra steps to iterate over multiple `mesh.json` files or parameterize the mesh name via `.env` values.
- **Observability** – append steps that push deployment metadata to your logging/monitoring stack.

When editing `deployMesh.yaml`, keep the secret-validation step up to date so failures happen quickly.

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
