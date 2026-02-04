# Documentation Specification

This is the documentation specification for the spec detailed in @.agent-os/specs/2026-02-03-reusable-cicd-workflows/spec.md

## Documentation Structure

```
/
├── README.md              # Project README
└── docs/
    └── user-guide.md      # Step-by-step guide for app repo initialization
```

---

## README.md (Project Root)

### Purpose
Explain what this repository is, what workflows it contains, and provide a quick reference.

### Required Sections

1. **Title & Description** - One-liner explaining the repo
2. **Workflows Overview** - Table of available workflows
3. **Quick Usage** - Minimal example of how app repos use it
4. **Architecture Diagram** - Visual of the CI/CD flow
5. **Links** - Link to full user guide in docs/

### Content Template

```markdown
# Reusable CI/CD Workflows

Centralized GitHub Actions workflows for `Startup-AI-Development` application repositories.

## Workflows

| Workflow | Purpose |
|----------|---------|
| `universal-ci.yaml` | Entry point - auto-detects language and dispatches |
| `nodejs-ci.yaml` | Node.js build and Docker push |
| `golang-ci.yaml` | Go build, test, and Docker push |
| `python-ci.yaml` | Python test and Docker push |
| `generic-ci.yaml` | Dockerfile-only build and push |
| `auto-version.yaml` | VERSION file bump on staging |

## Quick Usage

In your app repo, create `.github/workflows/ci.yaml`:

\`\`\`yaml
name: CI

on:
  push:
    branches: [develop, staging, main]
  pull_request:
    branches: [develop, staging, main]

jobs:
  ci:
    uses: Startup-AI-Infrastructure/apps-cicd-workflow-template/.github/workflows/universal-ci.yaml@main
    secrets: inherit
\`\`\`

## Versioning

| Branch | Tag Format | Example |
|--------|------------|---------|
| develop | `X.Y.Z-SNAPSHOT` | `1.0.0-SNAPSHOT` |
| staging | `X.Y.Z-RCn` | `1.0.0-RC1` |
| main | `X.Y.Z` | `1.0.0` |

## Documentation

See [docs/user-guide.md](docs/user-guide.md) for complete setup instructions.
```

---

## docs/user-guide.md

### Purpose
Complete step-by-step guide for developers to initialize a new app repository with CI/CD.

### Required Sections

1. **Prerequisites**
2. **Step 1: Create VERSION file**
3. **Step 2: Create Dockerfile**
4. **Step 3: Create k8s/ configuration**
5. **Step 4: Add CI workflow**
6. **Step 5: First deployment**
7. **Promotion flow** (develop → staging → main)
8. **Troubleshooting**

### Content Template

```markdown
# User Guide: Setting Up CI/CD for Your App

This guide explains how to configure your application repository to use the centralized CI/CD workflows.

## Prerequisites

- Repository in `Startup-AI-Development` organization
- Dockerfile in repo root
- Basic understanding of Git branching (develop/staging/main)

## Step 1: Create VERSION File

Create a `VERSION` file in your repo root with semantic version:

\`\`\`
1.0.0
\`\`\`

Rules:
- Format: `MAJOR.MINOR.PATCH`
- No `v` prefix
- Start with `1.0.0` for new projects

## Step 2: Create Dockerfile

Ensure you have a `Dockerfile` in your repo root. Example for Node.js:

\`\`\`dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
\`\`\`

## Step 3: Create k8s/ Configuration

Create the following files:

### k8s/global.yaml

\`\`\`yaml
preset: api  # api | worker | job | cronjob | daemon

image:
  repository: ghcr.io/startup-ai-development/YOUR-APP-NAME
  tag: "1.0.0-SNAPSHOT"
  pullPolicy: IfNotPresent

service:
  port: 3000

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: your-app.example.com
      paths:
        - path: /
          pathType: Prefix

probes:
  liveness:
    path: /health
    port: http
  readiness:
    path: /ready
    port: http
\`\`\`

### k8s/dev.yaml

\`\`\`yaml
replicaCount: 1

resources:
  requests:
    memory: 256Mi
    cpu: 100m
  limits:
    memory: 512Mi
    cpu: 500m

env:
  LOG_LEVEL: debug
\`\`\`

### k8s/stg.yaml

\`\`\`yaml
replicaCount: 2

resources:
  requests:
    memory: 512Mi
    cpu: 200m
  limits:
    memory: 1Gi
    cpu: 500m
\`\`\`

### k8s/prod.yaml

\`\`\`yaml
replicaCount: 3

resources:
  requests:
    memory: 512Mi
    cpu: 250m
  limits:
    memory: 1Gi
    cpu: 1000m

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
\`\`\`

## Step 4: Add CI Workflow

Create `.github/workflows/ci.yaml`:

\`\`\`yaml
name: CI

on:
  push:
    branches: [develop, staging, main]
  pull_request:
    branches: [develop, staging, main]

jobs:
  ci:
    uses: Startup-AI-Infrastructure/apps-cicd-workflow-template/.github/workflows/universal-ci.yaml@main
    secrets: inherit

  bump-version:
    needs: ci
    if: github.event_name == 'push' && github.ref_name == 'staging'
    uses: Startup-AI-Infrastructure/apps-cicd-workflow-template/.github/workflows/auto-version.yaml@main
    with:
      bump-type: minor
    secrets: inherit
\`\`\`

## Step 5: First Deployment

1. Commit all files to `develop` branch
2. Push to GitHub
3. CI will:
   - Detect your language
   - Build Docker image
   - Push to `ghcr.io/startup-ai-development/your-app:1.0.0-SNAPSHOT`
   - Update `k8s/global.yaml` with new tag
4. ArgoCD will automatically deploy to dev environment

## Promotion Flow

### develop → staging

\`\`\`bash
gh pr create --base staging --head develop --title "Promote to staging"
\`\`\`

On merge:
- VERSION bumps (1.0.0 → 1.1.0)
- Image tagged as `1.1.0-RC1`
- Deploys to staging environment

### staging → main

\`\`\`bash
gh pr create --base main --head staging --title "Release 1.1.0"
\`\`\`

On merge (requires approval):
- Image tagged as `1.1.0` (final release)
- Deploys to production environment

## Troubleshooting

### "Permission denied" pushing to ghcr.io

Check repository Settings → Actions → General → Workflow permissions:
- Enable "Read and write permissions"

### k8s/global.yaml not found

Ensure the file exists before first CI run. The workflow expects this file to update the image tag.

### VERSION file invalid

Must be plain semver without prefix:
- ✅ `1.0.0`
- ❌ `v1.0.0`

### Build not detecting my language

Detection order:
1. `package.json` → Node.js
2. `go.mod` → Go
3. `pyproject.toml` or `requirements.txt` → Python
4. Fallback → Generic (Dockerfile only)
```

---

## Tone and Style

- Direct, actionable instructions
- Code blocks for all file contents (copy-paste ready)
- Minimal explanation - focus on "how"
- Use English for docs (international team)
