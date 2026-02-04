# Spec Summary (Lite)

Add commit SHA tracking to CI workflow via `image.revision` field in per-environment k8s yaml files. This enables Kubernetes pod rollouts when the image tag stays the same but the underlying image changes. Also adds `[skip ci]` to manifest commits to prevent infinite CI loops.
