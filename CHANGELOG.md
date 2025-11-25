# Changelog

All notable changes to this project will be documented in this file. The format loosely follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the repository uses [Semantic Versioning](https://semver.org/).

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
