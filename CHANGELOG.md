# Changelog

All notable changes to this project will be documented in this file. The format loosely follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the repository uses [Semantic Versioning](https://semver.org/).

## [2.1.0] - 2026-09-15
### Changed
- Runner image now bakes in `@adobe/aio-cli-plugin-api-mesh`. `XDG_DATA_HOME` is pinned to an absolute path so oclif resolves the plugin regardless of the `$HOME` GitHub imposes on container jobs — the reason the plugin previously had to be reinstalled on every run.
- `deploy.yaml` no longer runs `aio plugins:install`, `actions/setup-node`, or the AIO debug step, and no longer overrides `HOME`. A single step asserts the baked-in CLI and plugin are present.
- Runner image base moved from `node:20-bullseye-slim` to `node:20-bookworm-slim`. Debian 11's security pool has been archived, so the old base no longer builds — `apt-get install git` 404s on packages its own index still lists. Node stays on 20.

### Added
- `.github/workflows/build-image.yaml` builds and pushes `ghcr.io/<owner>/aio-mesh-runner` on changes to the Dockerfile, tagging both `latest` and the commit SHA. Previously the image was only ever built by hand.
- Build-time assertion in the Dockerfile (`HOME=/tmp/homecheck aio api-mesh --help`) that fails the build if the plugin stops resolving independently of `$HOME`.

### Fixed
- `Select project` and `Select workspace` no longer hang and exit 130 when the Adobe organization has not accepted the Developer Terms of Service. Both commands call `ensureDevTermAccepted`, which prompts with no non-interactive flag available; the prompt is now answered on stdin.

## [2.0.1] - 2026-02-25
### Added
- New "Newman Testing Configuration" section in README documenting folder structure, file details, and step-by-step instructions for setting up Postman/Newman regression tests.
- New "How Environment Variable Injection Works" subsection explaining the `AIO_ENV_FILE` mechanism and `.env` materialization flow.
- Updated repository layout in README to include `tests/newman/` folder structure.

### Fixed
- Added explicit `shell: bash` to "Materialize branch .env file" step in `deploy.yaml` for consistency with other shell steps.

## [1.1.1] - 2025-12-04
### Added
- Workflow now consumes GitHub repository variables `ENV_STAGE` / `ENV_PROD`, writes their full multi-line contents into `.env`, and feeds that file to the Adobe API Mesh CLI before create/update operations.
- Documentation updates explaining the new `.env` injection path and the recommended GitHub variables for environment-specific configuration.

## [1.1.0] - 2025-11-25
### Added
- New primary workflow `.github/workflows/deploy.yaml` that materializes branch-specific mesh secrets, computes CLI flags automatically, and waits for mesh provisioning with retry logic.
- Reusable Newman regression workflow `.github/workflows/tests.yaml` plus a dependent `tests` job so deployments can trigger API smoke tests per branch.

### Changed
- Deployment job now emits `secrets.yaml` at runtime from encrypted GitHub secrets (`MESH_SECRETS_STAGE` / `MESH_SECRETS_PROD`) to keep sensitive resolver credentials out of the repository.
- Mesh commands (`aio api-mesh:create` / `update`) automatically include `--env .env` and `--secrets secrets.yaml` when those files exist, reducing manual flag drift.
- Mesh provisioning phase now polls `aio api-mesh:status` with clearer logging and early-failure handling if the mesh never reaches a healthy state.

## [1.0.0] - 2025-11-19
### Added
- Initial Adobe API Mesh CI/CD template with a GitHub Actions workflow for staging/production deployments, README-based setup guidance, and secret validation.

## [2.0.0] - 2025-12-09
### Added
- Prebuilt runner image with `aio`
- no CLI installation steps in the workflow.
