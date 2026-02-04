# Spec Tasks

## Tasks

- [x] 1. Refactor build job to produce artifacts
  - [x] 1.1 Separate build steps from Docker build in nodejs-ci.yaml
  - [x] 1.2 Add artifact upload step for node_modules/, dist/, package-lock.json
  - [x] 1.3 Set artifact retention to 1 day
  - [x] 1.4 Verify build job runs independently

- [x] 2. Implement unit test job
  - [x] 2.1 Create unit-tests job with `needs: build`
  - [x] 2.2 Add artifact download step
  - [x] 2.3 Add logic to detect if unit tests exist in tests/unit/
  - [x] 2.4 Run `npm test` if tests exist, skip gracefully if not
  - [ ] 2.5 Test with repo that has unit tests
  - [ ] 2.6 Test with repo that has no unit tests (verify graceful skip)

- [x] 3. Implement integration test job
  - [x] 3.1 Create integration-tests job with `needs: build`
  - [x] 3.2 Add artifact download step
  - [x] 3.3 Add logic to detect if integration tests exist in tests/integration/
  - [x] 3.4 Run `npm run test:integration` if tests exist, skip gracefully if not
  - [x] 3.5 Set job timeout to 10 minutes for testcontainers
  - [ ] 3.6 Test with repo that has integration tests (testcontainers)
  - [ ] 3.7 Test with repo that has no integration tests (verify graceful skip)

- [x] 4. Implement security audit job
  - [x] 4.1 Create security-audit job with `needs: build`
  - [x] 4.2 Add artifact download step (needs package-lock.json)
  - [x] 4.3 Run `npm audit --audit-level=high`
  - [ ] 4.4 Verify job fails on high/critical vulnerabilities
  - [ ] 4.5 Verify job passes when no high/critical vulnerabilities

- [x] 5. Add conditional Docker build gate
  - [x] 5.1 Update docker-build job to depend on all test jobs
  - [x] 5.2 Add branch condition: only develop, staging, main
  - [ ] 5.3 Verify Docker build is skipped on feature branches
  - [ ] 5.4 Verify Docker build runs on develop/staging/main after tests pass
  - [ ] 5.5 Verify Docker build is blocked when any test job fails

- [ ] 6. End-to-end validation
  - [ ] 6.1 Test complete workflow on feature branch (no Docker build)
  - [ ] 6.2 Test complete workflow on develop branch (full pipeline)
  - [x] 6.3 Update documentation if needed
  - [ ] 6.4 Verify all tests pass
