# Technical Specification

This is the technical specification for the spec detailed in @.agent-os/specs/2026-02-04-nodejs-ci-testing/spec.md

## Workflow Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ALL BRANCHES                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────┐                                                     │
│  │  Build  │ npm ci + npm run build                              │
│  └────┬────┘                                                     │
│       │ artifacts: node_modules/, dist/, package-lock.json       │
│       ▼                                                          │
│  ┌─────────────┬─────────────────┬──────────────┐               │
│  │ Unit Tests  │ Integration     │ Security     │  (parallel)   │
│  │ npm test    │ npm run         │ npm audit    │               │
│  │             │ test:integration│ --level=high │               │
│  └──────┬──────┴────────┬────────┴──────┬───────┘               │
│         │               │               │                        │
│         └───────────────┼───────────────┘                        │
│                         ▼                                        │
└─────────────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          │  ONLY: develop, staging, main │
          ├───────────────────────────────┤
          │  ┌─────────────────────────┐  │
          │  │     Docker Build        │  │
          │  │  build + push to GHCR   │  │
          │  └───────────────────────┬─┘  │
          │                          │    │
          │  ┌───────────────────────▼┐   │
          │  │     Update K8s        │   │
          │  │  update global.yaml   │   │
          │  └───────────────────────┘   │
          └───────────────────────────────┘
```

## Technical Requirements

### Job 1: Build

- **Runs on:** All branches (via workflow_call)
- **Node version:** Configurable input (default: 20)
- **Steps:**
  1. Checkout repository
  2. Setup Node.js with npm cache
  3. `npm ci` - deterministic install from package-lock.json
  4. `npm run build --if-present` - compile TypeScript
- **Artifacts uploaded:**
  - `node_modules/` - for test jobs to reuse
  - `dist/` - compiled output
  - `package-lock.json` - for audit and Docker build
- **Artifact retention:** 1 day (only needed for current pipeline run)

### Job 2: Unit Tests

- **Depends on:** build
- **Runs on:** All branches
- **Downloads artifacts:** node_modules/, dist/
- **Steps:**
  1. Download build artifacts
  2. Check if `tests/unit/` directory exists and contains test files
  3. If tests exist: run `npm test`
  4. If no tests: log message and exit successfully
- **Graceful handling:**
  ```bash
  if find tests/unit -name "*.test.ts" -o -name "*.test.js" 2>/dev/null | grep -q .; then
    npm test
  else
    echo "No unit tests found, skipping"
  fi
  ```

### Job 3: Integration Tests

- **Depends on:** build
- **Runs on:** All branches
- **Downloads artifacts:** node_modules/, dist/
- **Testcontainers support:** Requires Docker-in-Docker or privileged runner
- **Steps:**
  1. Download build artifacts
  2. Check if `tests/integration/` directory exists and contains test files
  3. If tests exist: run `npm run test:integration`
  4. If no tests: log message and exit successfully
- **Graceful handling:**
  ```bash
  if find tests/integration -name "*.integration.test.ts" -o -name "*.integration.test.js" 2>/dev/null | grep -q .; then
    npm run test:integration
  else
    echo "No integration tests found, skipping"
  fi
  ```
- **Timeout:** 10 minutes (testcontainers startup can be slow)

### Job 4: Security Audit

- **Depends on:** build
- **Runs on:** All branches
- **Downloads artifacts:** package-lock.json (and node_modules for npm context)
- **Steps:**
  1. Download build artifacts
  2. Run `npm audit --audit-level=high`
- **Failure behavior:** Fails job if high or critical vulnerabilities found
- **Note:** Only checks production dependencies by default

### Job 5: Docker Build (Conditional)

- **Depends on:** unit-tests, integration-tests, security-audit (all must pass)
- **Runs on:** Only `develop`, `staging`, `main` branches
- **Condition:**
  ```yaml
  if: github.ref_name == 'develop' || github.ref_name == 'staging' || github.ref_name == 'main'
  ```
- **Steps:**
  1. Checkout (fresh, not from artifacts)
  2. Setup Docker Buildx
  3. Login to GHCR
  4. Build and push multi-arch image
  5. Create git tag (staging/main only)

### Job 6: Update K8s (Conditional)

- **Depends on:** docker-build
- **Runs on:** Only `develop`, `staging`, `main` branches
- **Unchanged from current implementation**

## GitHub Actions Specifics

### Artifact Sharing Strategy

```yaml
# In build job
- uses: actions/upload-artifact@v4
  with:
    name: build-artifacts
    path: |
      node_modules/
      dist/
      package-lock.json
    retention-days: 1

# In test jobs
- uses: actions/download-artifact@v4
  with:
    name: build-artifacts
```

### Parallel Job Configuration

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    # ... build steps

  unit-tests:
    needs: build
    runs-on: ubuntu-latest
    # ... test steps

  integration-tests:
    needs: build
    runs-on: ubuntu-latest
    # ... test steps

  security-audit:
    needs: build
    runs-on: ubuntu-latest
    # ... audit steps

  docker-build:
    needs: [unit-tests, integration-tests, security-audit]
    if: contains(fromJSON('["develop", "staging", "main"]'), github.ref_name)
    runs-on: ubuntu-latest
    # ... docker steps
```

### Testcontainers in GitHub Actions

GitHub Actions runners support Docker natively, so testcontainers should work out of the box:

```yaml
integration-tests:
  runs-on: ubuntu-latest
  services: {} # No need for explicit services - testcontainers handles it
  steps:
    - uses: actions/download-artifact@v4
    - name: Run integration tests
      run: |
        # testcontainers uses the runner's Docker daemon
        npm run test:integration
```

## Input Parameters (workflow_call)

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `image-name` | string | yes | - | Full image name (ghcr.io/org/repo) |
| `image-tag` | string | yes | - | Image tag to use |
| `push-image` | boolean | no | true | Whether to push the image |
| `node-version` | string | no | '20' | Node.js version to use |
| `skip-tests` | boolean | no | false | Skip test jobs (emergency bypass) |

## Error Handling

- **Build failure:** Entire pipeline stops, no tests run
- **Unit test failure:** Docker build blocked
- **Integration test failure:** Docker build blocked
- **Security audit failure:** Docker build blocked
- **No tests found:** Job succeeds with informational message
- **Docker build failure:** K8s update skipped
