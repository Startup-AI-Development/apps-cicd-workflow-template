# Spec Requirements Document

> Spec: Commit Revision Tracking
> Created: 2026-02-04

## Overview

Add commit SHA tracking to the CI workflow to enable Kubernetes pod rollouts when the image tag remains unchanged but the underlying image changes. This ensures deployments always reflect the latest build by using a revision annotation that triggers pod recreation.

## User Stories

### Automatic Pod Rollout on New Builds

As a platform engineer, I want Kubernetes deployments to automatically roll out when a new image is pushed, so that the cluster always runs the latest code even when the image tag (e.g., `1.0.0-SNAPSHOT`) doesn't change.

Currently, when pushing to `develop` with the same `1.0.0-SNAPSHOT` tag, Kubernetes sees no change in the deployment spec and skips the rollout. By adding a `revision` field with the commit SHA, each build produces a unique deployment spec that triggers a rolling update.

### Prevent CI Infinite Loops

As a developer, I want CI commits to not trigger additional CI runs, so that updating k8s manifests doesn't create an infinite loop of builds.

The workflow commits changes to k8s/*.yaml files. Without `[skip ci]`, this commit triggers another CI run, which commits again, creating an infinite loop.

## Spec Scope

1. **Commit SHA Output** - Add `commit-sha` output to the `detect-and-version` job containing the short (7-char) commit hash
2. **Per-Environment Revision Update** - Update the environment-specific yaml file (`dev.yaml`, `stg.yaml`, `prod.yaml`) with `image.revision` based on branch
3. **Skip CI on Manifest Commits** - Add `[skip ci]` to the commit message to prevent infinite CI loops
4. **Documentation for Consuming Repos** - Document the Helm chart changes required in consuming repositories

## Out of Scope

- Modifications to the Helm chart template repository (separate spec)
- Changes to the ArgoCD/GitOps deployment configuration
- Rollback functionality based on revision

## Expected Deliverable

1. CI workflow pushes to `develop` and the `k8s/dev.yaml` file contains `image.revision: <commit-sha>`
2. CI workflow commits with `[skip ci]` do not trigger additional workflow runs
3. Documentation exists explaining how consuming repos should configure their Helm charts to use `image.revision`
