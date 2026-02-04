# Spec Tasks

## Tasks

- [x] 1. Add commit-sha output to detect-and-version job
  - [x] 1.1 Add `commit-sha` to job outputs section (line ~47)
  - [x] 1.2 Add `echo "commit-sha=${GITHUB_SHA::7}"` to version step (line ~109)
  - [x] 1.3 Verify output is accessible in subsequent jobs

- [x] 2. Implement per-environment revision update in update-k8s job
  - [x] 2.1 Add branch-to-file mapping logic (develop→dev.yaml, staging→stg.yaml, main→prod.yaml)
  - [x] 2.2 Add conditional check for environment file existence
  - [x] 2.3 Add yq command to update `image.revision` in environment file
  - [x] 2.4 Update git add to include modified environment file

- [x] 3. Add [skip ci] to prevent infinite loops
  - [x] 3.1 Modify commit message to include `[skip ci]` suffix
  - [x] 3.2 Verify CI does not trigger on manifest update commits

- [x] 4. Create documentation for consuming repositories
  - [x] 4.1 Document Helm chart annotation requirement (`app.kubernetes.io/revision`)
  - [x] 4.2 Document values.yaml structure with `image.revision` field
  - [x] 4.3 Add documentation to README or docs folder
