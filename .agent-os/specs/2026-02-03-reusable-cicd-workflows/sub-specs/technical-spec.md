# Technical Specification

This is the technical specification for the spec detailed in @.agent-os/specs/2026-02-03-reusable-cicd-workflows/spec.md

## Workflow Architecture

```
.github/workflows/
├── universal-ci.yaml      # Entry point - detects language, dispatches to specific workflow
├── nodejs-ci.yaml         # Node.js: npm install → docker build/push (tests omitted)
├── golang-ci.yaml         # Go: go mod download → go test → docker build/push
├── python-ci.yaml         # Python: pip install → pytest → docker build/push
├── generic-ci.yaml        # Generic: docker build/push only
└── auto-version.yaml      # Bumps VERSION file on staging merges
```

## Technical Requirements

### 1. Language Detection Logic (universal-ci.yaml)

Detection order (first match wins):
1. **Node.js**: `package.json` exists in repo root
2. **Go**: `go.mod` exists in repo root
3. **Python**: `pyproject.toml` OR `requirements.txt` exists in repo root
4. **Generic**: Fallback when no language files detected (requires `Dockerfile`)

Implementation:
```yaml
- name: Detect language
  id: detect
  run: |
    if [ -f "package.json" ]; then
      echo "language=nodejs" >> $GITHUB_OUTPUT
    elif [ -f "go.mod" ]; then
      echo "language=golang" >> $GITHUB_OUTPUT
    elif [ -f "pyproject.toml" ] || [ -f "requirements.txt" ]; then
      echo "language=python" >> $GITHUB_OUTPUT
    else
      echo "language=generic" >> $GITHUB_OUTPUT
    fi
```

### 2. Versioning Strategy

| Branch | Tag Format | Example | Mutable |
|--------|------------|---------|---------|
| `develop` | `{VERSION}-SNAPSHOT` | `1.2.0-SNAPSHOT` | Yes |
| `staging` | `{VERSION}-RC{n}` | `1.2.0-RC1`, `1.2.0-RC2` | No |
| `main` | `{VERSION}` | `1.2.0` | No |

**RC Counter Logic:**
- Count existing tags matching `{VERSION}-RC*` pattern
- Increment by 1 for new RC
- Store in git tag, not in a file

```yaml
- name: Determine RC number
  if: github.ref_name == 'staging'
  run: |
    VERSION=$(cat VERSION)
    EXISTING_RCS=$(git tag -l "${VERSION}-RC*" | wc -l)
    RC_NUM=$((EXISTING_RCS + 1))
    echo "IMAGE_TAG=${VERSION}-RC${RC_NUM}" >> $GITHUB_ENV
```

### 3. Docker Image Naming

**Repository**: `ghcr.io/startup-ai-development/{app-name}`

The `{app-name}` is derived from `${{ github.repository }}` which gives `Startup-AI-Development/app-name`.

```yaml
- name: Set image name
  run: |
    REPO_NAME=$(echo "${{ github.repository }}" | cut -d'/' -f2)
    echo "IMAGE_NAME=ghcr.io/startup-ai-development/${REPO_NAME}" >> $GITHUB_ENV
```

### 4. Workflow Call Interface

Each language workflow must accept these inputs:

```yaml
on:
  workflow_call:
    inputs:
      image-name:
        description: 'Full image name (ghcr.io/org/repo)'
        required: true
        type: string
      image-tag:
        description: 'Image tag to use'
        required: true
        type: string
      push-image:
        description: 'Whether to push the image'
        required: false
        type: boolean
        default: true
```

### 5. k8s/global.yaml Update

After successful image push, update the `image.tag` field:

```yaml
- name: Update k8s/global.yaml
  if: github.event_name == 'push'  # Not on PRs
  run: |
    # Update image.tag using yq
    yq -i '.image.tag = "${{ env.IMAGE_TAG }}"' k8s/global.yaml

    # Commit and push
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add k8s/global.yaml
    git commit -m "ci: update image tag to ${{ env.IMAGE_TAG }}"
    git push
```

### 6. GITHUB_TOKEN Permissions

Required permissions for all workflows:

```yaml
permissions:
  contents: write      # For committing k8s/global.yaml changes
  packages: write      # For pushing to ghcr.io
  pull-requests: read  # For PR context
```

### 7. Language-Specific Build Steps

#### Node.js (nodejs-ci.yaml)
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
- run: npm ci
- run: npm run build --if-present
# Note: npm test omitted for now - can be added later
```

#### Go (golang-ci.yaml)
```yaml
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'
- run: go mod download
- run: go test ./...
- run: go build -o app .
```

#### Python (python-ci.yaml)
```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
- run: |
    if [ -f "pyproject.toml" ]; then
      pip3 install -e ".[dev]"
    else
      pip3 install -r requirements.txt
      pip3 install pytest
    fi
- run: pytest
```

### 8. Docker Build (Common to all)

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: ${{ inputs.push-image }}
    tags: ${{ inputs.image-name }}:${{ inputs.image-tag }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### 9. auto-version.yaml Behavior

Called separately after CI succeeds on staging:

```yaml
on:
  workflow_call:
    inputs:
      bump-type:
        description: 'Version bump type (major, minor, patch)'
        required: false
        type: string
        default: 'minor'
```

Logic:
1. Read current VERSION file
2. Parse semver (X.Y.Z)
3. Increment specified component
4. Write back to VERSION
5. Commit and push

### 10. PR vs Push Behavior

| Event | Build | Test* | Push Image | Update k8s | Create Tag |
|-------|-------|-------|------------|------------|------------|
| PR | Yes | Yes* | No | No | No |
| Push to develop | Yes | Yes* | Yes | Yes | No |
| Push to staging | Yes | Yes* | Yes | Yes | Yes (RC) |
| Push to main | Yes | Yes* | Yes | Yes | Yes (release) |

*Tests: Go and Python run tests; Node.js tests omitted for now

## Error Handling

- If tests fail: Stop workflow, do not build/push image
- If Docker build fails: Stop workflow, report error
- If k8s/global.yaml update fails: Report warning but don't fail workflow (image already pushed)
- If no Dockerfile exists: Fail workflow with clear error message

## Concurrency

Prevent concurrent runs on the same branch:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```
