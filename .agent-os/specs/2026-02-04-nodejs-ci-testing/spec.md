# Spec Requirements Document

> Spec: Node.js CI Testing Pipeline
> Created: 2026-02-04

## Overview

Refactor the Node.js CI workflow to separate build and test phases, adding parallel unit tests, integration tests, and security audit jobs that must pass before Docker image building. This improves code quality assurance and security posture while maintaining CI performance through parallelization.

## User Stories

### Reliable Build Artifacts

As a developer, I want the CI pipeline to produce consistent build artifacts (package-lock.json and dist/), so that subsequent jobs use the exact same dependencies and compiled code.

The build job runs `npm ci` to install dependencies deterministically from package-lock.json, then runs `npm run build` to compile TypeScript. These artifacts are uploaded and shared with downstream jobs to ensure consistency.

### Parallel Test Execution

As a developer, I want unit and integration tests to run in parallel after the build, so that I get fast feedback on code quality without blocking on sequential test execution.

Both test jobs download the build artifacts, then execute their respective test suites. If no tests exist in the expected folders (`tests/unit/` or `tests/integration/`), the job completes successfully rather than failing the pipeline.

### Security Vulnerability Detection

As a security-conscious developer, I want the CI to check for high and critical npm vulnerabilities, so that insecure dependencies are caught before deployment.

The security audit job runs `npm audit --audit-level=high` in parallel with tests. If high or critical vulnerabilities are found, the job fails and blocks the Docker build.

## Spec Scope

1. **Build Job Refactor** - Separate build phase that produces package-lock.json and dist/ as uploadable artifacts (runs on all branches)
2. **Unit Test Job** - Parallel job running `npm test` with graceful handling when no unit tests exist (runs on all branches)
3. **Integration Test Job** - Parallel job running `npm run test:integration` with testcontainers support and graceful handling when no integration tests exist (runs on all branches)
4. **Security Audit Job** - Parallel job running `npm audit --audit-level=high` to detect vulnerabilities (runs on all branches)
5. **Docker Build Gate** - Docker build only proceeds if all three parallel jobs pass, and only runs on `develop`, `staging`, `main` branches

## Out of Scope

- Automatic vulnerability fixing (`npm audit fix`)
- Code coverage enforcement or reporting
- E2E testing
- Performance/load testing
- Notification integrations (Slack, email)
- Custom test reporters

## Expected Deliverable

1. Modified `nodejs-ci.yaml` workflow with build job producing artifacts and three parallel test/audit jobs
2. Docker build job that depends on all test jobs passing
3. Test jobs that succeed gracefully when no tests exist in expected folders
