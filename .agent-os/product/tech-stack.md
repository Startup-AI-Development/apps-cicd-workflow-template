# Technical Stack

## CI/CD Platform

- **Platform:** GitHub Actions
- **Workflow Type:** Reusable Workflows (called via `uses:`)
- **Runner:** GitHub-hosted runners (ubuntu-latest)

## Container Infrastructure

- **Container Registry:** ghcr.io (GitHub Container Registry)
- **Image Build:** Docker with multi-stage builds
- **Build Tool:** docker/build-push-action

## Supported Languages

- **Node.js:** Detection via package.json, npm/yarn builds
- **Go:** Detection via go.mod, go build/test
- **Python:** Detection via requirements.txt/pyproject.toml, pip/poetry
- **Generic:** Fallback for Dockerfile-only projects

## Versioning Strategy

| Branch | Tag Format | Example |
|--------|------------|---------|
| develop | `SNAPSHOT-{sha}` | `SNAPSHOT-abc1234` |
| staging | `RC-{version}` | `RC-1.2.3` |
| main | `{version}` | `1.2.3` |

## Kubernetes Integration

- **Manifest Format:** YAML (k8s/global.yaml)
- **Update Method:** yq or sed-based tag replacement
- **Target Field:** `image.tag` in values/config

## Authentication & Secrets

- **Registry Auth:** GITHUB_TOKEN (automatic)
- **Manifest Repo Access:** Deploy key or PAT (configurable)

## Code Repository

- **URL:** https://github.com/Startup-AI-Infrastructure/apps-cicd-workflow-template
- **Hosting:** GitHub
