# Spec Requirements Document

> Spec: Reusable CI/CD Workflows
> Created: 2026-02-03

## Overview

Implement a complete set of reusable GitHub Actions workflows that application repositories in `Startup-AI-Development` can call via `uses:` to automate build, test, Docker image creation, semantic versioning based on branch, and automatic k8s/global.yaml image tag updates.

## User Stories

### Developer Builds and Deploys Without Writing CI/CD

As a developer, I want to add a minimal CI workflow file to my app repo that calls the centralized reusable workflows, so that my application automatically builds, tests, and deploys without me writing complex CI/CD logic.

When I push code to `develop`, the workflow detects my project language (Node.js, Go, Python, or generic Dockerfile), runs the appropriate tests, builds a Docker image, tags it as `X.Y.Z-SNAPSHOT`, pushes it to ghcr.io, and updates my k8s/global.yaml with the new tag.

### Release Candidate Creation on Staging Merge

As a team lead, I want merges to the `staging` branch to automatically create immutable release candidate tags (`X.Y.Z-RC1`, `RC2`, etc.), so that I can validate specific versions before production release.

When I merge to `staging`, the workflow bumps the VERSION file (if configured), creates an immutable RC tag, and updates k8s/global.yaml so ArgoCD deploys the new version to staging.

### Production Release on Main Merge

As a release manager, I want merges to `main` to create clean release tags (`X.Y.Z`) without any suffix, so that production deployments use clearly versioned, immutable images.

When I merge to `main`, the workflow creates the final release tag, pushes the image, and updates k8s/global.yaml for production deployment.

## Spec Scope

1. **universal-ci.yaml** - Entry point workflow that detects project language and dispatches to the appropriate language-specific workflow
2. **nodejs-ci.yaml** - Reusable workflow for Node.js projects (npm install, Docker build/push - tests omitted for now)
3. **golang-ci.yaml** - Reusable workflow for Go projects (go mod download, go test, Docker build/push)
4. **python-ci.yaml** - Reusable workflow for Python projects (pip install, pytest, Docker build/push)
5. **generic-ci.yaml** - Reusable workflow for Dockerfile-only projects (Docker build/push only)
6. **auto-version.yaml** - Workflow for automatic VERSION file bumping on staging merges
7. **README.md** - Project README explaining what this repo contains and how it works
8. **docs/user-guide.md** - Step-by-step guide for initializing a new app repository to use these workflows

## Out of Scope

- ArgoCD configuration (already exists in gitops-k8s-manifests)
- Helm chart development (already exists in apps-helm-chart-template)
- Deployment monitoring and alerting
- Custom build tooling beyond standard language tools (npm, go, pip3)
- Multi-architecture Docker builds (arm64, etc.)
- Custom secrets management beyond GITHUB_TOKEN

## Expected Deliverable

1. All 6 workflow files created in `.github/workflows/` directory, each using `on: workflow_call` trigger
2. App repos can call `universal-ci.yaml` with `uses: Startup-AI-Infrastructure/apps-cicd-workflow-template/.github/workflows/universal-ci.yaml@main` and have their CI/CD fully automated
3. Docker images are pushed to `ghcr.io/Startup-AI-Development/{app-name}` with correct branch-based versioning (SNAPSHOT/RC/release)
4. k8s/global.yaml `image.tag` field is automatically updated and committed after successful image push
5. Project `README.md` documenting the repository purpose and workflow overview
6. User guide in `docs/user-guide.md` with step-by-step instructions for initializing new app repositories
