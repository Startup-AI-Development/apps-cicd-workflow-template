# Technical Specification

This is the technical specification for the spec detailed in @.agent-os/specs/2026-02-04-commit-revision-tracking/spec.md

## Technical Requirements

### 1. Add `commit-sha` Output (detect-and-version job)

**Location:** `universal-ci.yaml` line ~47 (outputs section)

```yaml
outputs:
  language: ${{ steps.detect.outputs.language }}
  image-name: ${{ steps.version.outputs.image-name }}
  image-tag: ${{ steps.version.outputs.image-tag }}
  commit-sha: ${{ steps.version.outputs.commit-sha }}  # NEW
```

**Location:** `universal-ci.yaml` line ~109 (version step)

```bash
echo "commit-sha=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT
```

### 2. Branch-to-File Mapping for Revision Updates

The workflow must map branches to their corresponding environment files:

| Branch | Environment File |
|--------|------------------|
| `develop` | `k8s/dev.yaml` |
| `staging` | `k8s/stg.yaml` |
| `main` | `k8s/prod.yaml` |

### 3. Update Environment File with Tag and Revision (update-k8s job)

**Location:** `universal-ci.yaml` update-k8s job

The workflow updates the per-environment file with both `image.tag` and `image.revision`:

```bash
# Determine environment file based on branch
BRANCH="${{ github.ref_name }}"
case "$BRANCH" in
  develop)
    ENV_FILE="k8s/dev.yaml"
    ;;
  staging)
    ENV_FILE="k8s/stg.yaml"
    ;;
  main)
    ENV_FILE="k8s/prod.yaml"
    ;;
  *)
    ENV_FILE=""
    ;;
esac

# Update image.tag and image.revision in environment file (creates fields if not exist)
if [ -n "$ENV_FILE" ] && [ -f "$ENV_FILE" ]; then
  yq -i '.image.tag = "${{ needs.detect-and-version.outputs.image-tag }}"' "$ENV_FILE"
  yq -i '.image.revision = "${{ needs.detect-and-version.outputs.commit-sha }}"' "$ENV_FILE"
fi
```

**Note:** `k8s/global.yaml` is NOT modified. All image-related values go in per-environment files.

### 4. Add [skip ci] to Commit Message

**Location:** `universal-ci.yaml` line ~402 (commit step)

```bash
git commit -m "ci: update image tag to ${{ needs.detect-and-version.outputs.image-tag }} [skip ci]"
```

### 5. Update git add to Include Environment Files

**Location:** `universal-ci.yaml` line ~396 (git add step)

```bash
git add k8s/global.yaml
# Add environment file if it was modified
if [ -n "$ENV_FILE" ] && [ -f "$ENV_FILE" ]; then
  git add "$ENV_FILE"
fi
```

## Consuming Repository Requirements

Repositories using this workflow must configure their Helm charts as follows:

### Deployment Template Annotation

In the Helm chart deployment template, add:

```yaml
spec:
  template:
    metadata:
      annotations:
        app.kubernetes.io/revision: "{{ .Values.image.revision | default \"\" }}"
```

### Values.yaml Structure

```yaml
image:
  repository: ghcr.io/org/app
  tag: "1.0.0-SNAPSHOT"
  revision: ""  # Set automatically by CI
  pullPolicy: Always  # Recommended for dev/stg
```

## Edge Cases

1. **New repo without revision field**: `yq -i` creates the field automatically
2. **Feature branches**: No environment file mapping, revision update skipped
3. **Missing environment file**: Check existence before updating
4. **Empty environment file**: `yq` handles this, creates `image.revision` structure
