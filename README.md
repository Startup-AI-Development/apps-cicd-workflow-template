# Reusable CI/CD Workflows

Centralized GitHub Actions workflows that provide language-aware, branch-based CI/CD automation for application repositories.

## Workflows

| Workflow | Description | Use Case |
|----------|-------------|----------|
| `universal-ci.yaml` | Entry point with language detection | Default choice for any app |
| `nodejs-ci.yaml` | Node.js build (npm) + Docker | Node.js applications |
| `golang-ci.yaml` | Go build + test + Docker | Go applications |
| `python-ci.yaml` | Python install + pytest + Docker | Python applications |
| `generic-ci.yaml` | Docker build only | Apps without language-specific build |
| `auto-version.yaml` | Bump VERSION file | After staging merges |

## Quick Start

1. Create a `VERSION` file in your repo root:
   ```
   1.0.0
   ```

2. Create a `Dockerfile` in your repo root

3. Create `.github/workflows/ci.yaml`:
   ```yaml
   name: CI

   on:
     push:
       branches: [develop, staging, main]
     pull_request:
       branches: [develop, staging, main]

   jobs:
     ci:
       uses: Startup-AI-Development/apps-cicd-workflow-template/.github/workflows/universal-ci.yaml@main
       with:
         push-image: ${{ github.event_name == 'push' }}
   ```

4. Create `k8s/global.yaml` (optional, for auto-updating image tags):
   ```yaml
   image:
     tag: "1.0.0"
   ```

## Versioning

| Branch | Tag Format | Example | Mutable |
|--------|------------|---------|---------|
| `develop` | `{VERSION}-SNAPSHOT` | `1.0.0-SNAPSHOT` | Yes |
| `staging` | `{VERSION}-RC{n}` | `1.0.0-RC1` | No |
| `main` | `{VERSION}` | `1.0.0` | No |
| Other | `{VERSION}-{branch}` | `1.0.0-feature-xyz` | Yes |

## Features

- **Multi-arch builds**: Images are built for both `linux/amd64` and `linux/arm64`
- **Automatic language detection**: Detects Node.js, Go, Python, or falls back to generic Docker build
- **Branch-based versioning**: SNAPSHOT for develop, RC for staging, release for main
- **k8s manifest updates**: Automatically updates `k8s/global.yaml` with new image tags

## Language Detection

The `universal-ci.yaml` workflow automatically detects your project's language:

1. **Node.js**: `package.json` exists
2. **Go**: `go.mod` exists
3. **Python**: `pyproject.toml` or `requirements.txt` exists
4. **Generic**: Fallback (requires Dockerfile)

## Documentation

See the [User Guide](docs/user-guide.md) for detailed setup instructions.
