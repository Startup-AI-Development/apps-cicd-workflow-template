# Spec Tasks

## Tasks

- [x] 1. **Create generic-ci.yaml** - Base workflow with Docker build/push and k8s update
  - [x] 1.1 Create `.github/workflows/generic-ci.yaml` with `workflow_call` trigger
  - [x] 1.2 Implement versioning logic (SNAPSHOT/RC/release based on branch)
  - [x] 1.3 Add Docker build and push to ghcr.io
  - [x] 1.4 Add k8s/global.yaml image.tag update and commit
  - [x] 1.5 Add concurrency control

- [x] 2. **Create nodejs-ci.yaml** - Node.js specific workflow
  - [x] 2.1 Create `.github/workflows/nodejs-ci.yaml` with `workflow_call` trigger
  - [x] 2.2 Add Node.js setup with npm cache
  - [x] 2.3 Add `npm ci` and `npm run build`
  - [x] 2.4 Reuse Docker build/push and k8s update logic from generic

- [x] 3. **Create golang-ci.yaml** - Go specific workflow
  - [x] 3.1 Create `.github/workflows/golang-ci.yaml` with `workflow_call` trigger
  - [x] 3.2 Add Go setup with module cache
  - [x] 3.3 Add `go mod download`, `go test ./...`, `go build`
  - [x] 3.4 Reuse Docker build/push and k8s update logic

- [x] 4. **Create python-ci.yaml** - Python specific workflow
  - [x] 4.1 Create `.github/workflows/python-ci.yaml` with `workflow_call` trigger
  - [x] 4.2 Add Python setup
  - [x] 4.3 Add pip3 install (detect pyproject.toml vs requirements.txt)
  - [x] 4.4 Add pytest execution
  - [x] 4.5 Reuse Docker build/push and k8s update logic

- [x] 5. **Create universal-ci.yaml** - Entry point with language detection
  - [x] 5.1 Create `.github/workflows/universal-ci.yaml` with `workflow_call` trigger
  - [x] 5.2 Implement language detection logic (package.json → nodejs, go.mod → golang, etc.)
  - [x] 5.3 Add conditional job dispatch to appropriate language workflow
  - [x] 5.4 Pass through image-name, image-tag, and push-image inputs

- [x] 6. **Create auto-version.yaml** - VERSION file bump workflow
  - [x] 6.1 Create `.github/workflows/auto-version.yaml` with `workflow_call` trigger
  - [x] 6.2 Add input for bump-type (major, minor, patch)
  - [x] 6.3 Implement semver parsing and increment logic
  - [x] 6.4 Commit and push updated VERSION file

- [x] 7. **Create README.md** - Project documentation
  - [x] 7.1 Write project description and workflow overview table
  - [x] 7.2 Add quick usage example
  - [x] 7.3 Add versioning table and link to user guide

- [x] 8. **Create docs/user-guide.md** - Complete initialization guide
  - [x] 8.1 Create `docs/` directory
  - [x] 8.2 Write prerequisites section
  - [x] 8.3 Write step-by-step initialization (VERSION, Dockerfile, k8s/, ci.yaml)
  - [x] 8.4 Write promotion flow section (develop → staging → main)
  - [x] 8.5 Write troubleshooting section
