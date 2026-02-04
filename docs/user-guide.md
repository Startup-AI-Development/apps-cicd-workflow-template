# User Guide

Complete guide to setting up CI/CD for your application repository using these reusable workflows.

## Prerequisites

Before you begin, ensure:

- Your repository is hosted on GitHub under the `Startup-AI-Infrastructure` or `Startup-AI-Development` organization
- You have write access to the repository
- Your application has a `Dockerfile` in the repository root

## Step-by-Step Setup

### Step 1: Create VERSION File

Create a `VERSION` file in your repository root containing your initial version:

```
1.0.0
```

This file controls your application's semantic version. The CI workflow reads this file to generate image tags.

### Step 2: Create Dockerfile

Create a `Dockerfile` in your repository root. Example for Node.js:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
CMD ["node", "index.js"]
```

Example for Go:

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN go build -o app .

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/app .
CMD ["./app"]
```

Example for Python:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```

### Step 3: Create k8s Directory (Optional)

If you want the CI to automatically update your Kubernetes manifests with new image tags, create:

```
k8s/
├── global.yaml   # Shared values (namespace, probes, resources, etc.)
├── dev.yaml      # Development: image.tag + image.revision
├── stg.yaml      # Staging: image.tag + image.revision
└── prod.yaml     # Production: image.tag + image.revision
```

The per-environment files can start empty or with environment-specific overrides. The CI will automatically add `image.tag` and `image.revision` based on the branch:

| Branch | File Updated | Example Tag |
|--------|--------------|-------------|
| `develop` | `k8s/dev.yaml` | `1.0.0-SNAPSHOT` |
| `staging` | `k8s/stg.yaml` | `1.0.0-RC1` |
| `main` | `k8s/prod.yaml` | `1.0.0` |

After each successful image push, the workflow updates the environment-specific file with:
- `image.tag` - the version tag (e.g., `1.0.0-SNAPSHOT`)
- `image.revision` - the 7-char commit SHA for pod rollouts

### Step 4: Create CI Workflow

Create `.github/workflows/ci.yaml`:

```yaml
name: CI

on:
  push:
    branches: [develop, staging, main]
  pull_request:
    branches: [develop, staging, main]

jobs:
  ci:
    uses: Startup-AI-Infrastructure/apps-cicd-workflow-template/.github/workflows/universal-ci.yaml@main
    with:
      push-image: ${{ github.event_name == 'push' }}
```

This configuration:
- Runs on push to develop, staging, and main branches
- Runs on PRs targeting those branches
- Only pushes images on actual pushes (not on PRs)

### Step 5: Configure Repository Settings

1. Go to repository Settings > Actions > General
2. Under "Workflow permissions", select "Read and write permissions"
3. Check "Allow GitHub Actions to create and approve pull requests" (if using auto-version)

## Promotion Flow

### develop -> staging -> main

1. **Development** (`develop` branch)
   - Push to develop triggers CI
   - Image tagged as `1.0.0-SNAPSHOT`
   - SNAPSHOT tags are mutable (overwritten on each push)

2. **Staging** (`staging` branch)
   - Merge develop into staging
   - Image tagged as `1.0.0-RC1`, `1.0.0-RC2`, etc.
   - RC tags are immutable (incrementing counter)
   - Git tag created for each RC

3. **Production** (`main` branch)
   - Merge staging into main
   - Image tagged as `1.0.0`
   - Release tags are immutable
   - Git tag created for release

### Bumping Version

After releasing to production, bump the VERSION file for the next development cycle:

**Option A: Manual**
```bash
echo "1.1.0" > VERSION
git add VERSION
git commit -m "chore: bump version to 1.1.0"
git push
```

**Option B: Auto-version Workflow**
Create `.github/workflows/bump-version.yaml`:

```yaml
name: Bump Version

on:
  workflow_dispatch:
    inputs:
      bump-type:
        description: 'Version bump type'
        required: true
        default: 'minor'
        type: choice
        options:
          - major
          - minor
          - patch

jobs:
  bump:
    uses: Startup-AI-Infrastructure/apps-cicd-workflow-template/.github/workflows/auto-version.yaml@main
    with:
      bump-type: ${{ inputs.bump-type }}
```

Then trigger manually from Actions tab when needed.

## Revision Tracking for Pod Rollouts

When using mutable tags like `1.0.0-SNAPSHOT` (develop branch), Kubernetes won't automatically roll out new pods when you push a new image with the same tag. The `image.revision` field solves this by providing a unique value (commit SHA) for each build.

### How It Works

1. CI builds and pushes the image with tag `1.0.0-SNAPSHOT`
2. CI updates `k8s/dev.yaml` with `image.revision: abc1234`
3. Your Helm chart uses this revision as a pod annotation
4. When the revision changes, Kubernetes sees a spec change and triggers a rollout

### Helm Chart Configuration

To enable revision-based rollouts, configure your Helm chart:

**values.yaml:**
```yaml
image:
  repository: ghcr.io/your-org/your-app
  tag: "1.0.0-SNAPSHOT"
  revision: ""  # Set automatically by CI
  pullPolicy: Always  # Recommended for dev/stg
```

**deployment.yaml template:**
```yaml
spec:
  template:
    metadata:
      annotations:
        app.kubernetes.io/revision: "{{ .Values.image.revision | default "" }}"
```

The annotation ensures that any change to `image.revision` triggers a pod rollout, even when the image tag stays the same.

## Language-Specific Notes

### Node.js

- Detected by: `package.json` in root
- Build: `npm ci` + `npm run build --if-present`
- Tests: Currently skipped (can be added later)
- Node version: 20 (configurable)

### Go

- Detected by: `go.mod` in root
- Build: `go mod download` + `go build`
- Tests: `go test ./...` (required to pass)
- Go version: 1.22 (configurable)

### Python

- Detected by: `pyproject.toml` or `requirements.txt` in root
- Install: `pip3 install -e ".[dev]"` (pyproject) or `pip3 install -r requirements.txt` (requirements)
- Tests: `pytest` (required to pass)
- Python version: 3.12 (configurable)

### Generic

- Fallback when no language detected
- Requires `Dockerfile` only
- No build or test steps

## Troubleshooting

### "VERSION file not found"

Create a `VERSION` file in your repository root containing a valid semver (e.g., `1.0.0`).

### "Dockerfile not found"

Create a `Dockerfile` in your repository root. This is required for all workflows.

### Image push fails with 403

1. Check that GitHub Actions has write access to packages:
   - Settings > Actions > General > Workflow permissions > Read and write
2. Ensure the repository is in the correct organization

### k8s/global.yaml not updating

- The file must exist before the workflow runs
- The workflow only updates on push events (not PRs)
- Check the workflow logs for any error messages

### RC counter keeps incrementing unexpectedly

RC numbers are based on existing git tags matching the pattern `{VERSION}-RC*`. If you:
- Manually created tags
- Have tags from previous versions

You may see unexpected RC numbers. To reset, delete unwanted RC tags:
```bash
git tag -d 1.0.0-RC1
git push origin :refs/tags/1.0.0-RC1
```

### Tests failing in CI but passing locally

- Ensure all dev dependencies are in `pyproject.toml` or `requirements.txt`
- For Go: ensure `go.sum` is committed
- For Node.js: ensure `package-lock.json` is committed
- Check for environment-specific issues (paths, env vars)

### Concurrent workflow runs causing issues

The workflows include concurrency controls that cancel in-progress runs when new commits are pushed. If you need to run multiple workflows simultaneously, you may need to adjust the concurrency group in your local ci.yaml.
