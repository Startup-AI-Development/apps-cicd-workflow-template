# Spec Summary (Lite)

Implement reusable GitHub Actions workflows (`universal-ci.yaml`, `nodejs-ci.yaml`, `golang-ci.yaml`, `python-ci.yaml`, `generic-ci.yaml`, `auto-version.yaml`) that app repos call via `uses:` to automate language detection, build, Docker image push to ghcr.io with branch-based semantic versioning (develop=SNAPSHOT, staging=RC, main=release), and automatic k8s/global.yaml tag updates. Includes project `README.md` and `docs/user-guide.md` with step-by-step instructions for initializing new app repositories. Note: Node.js tests omitted for now; Go and Python include tests.
