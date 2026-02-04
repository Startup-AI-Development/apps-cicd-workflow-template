# Product Mission (Lite)

Reusable CI/CD Workflows is a GitHub Actions workflow library that helps development teams standardize their build, test, and deployment pipelines by providing language-aware, branch-based automation that pushes versioned container images and automatically updates Kubernetes manifests.

This repository serves development and platform engineering teams who need consistent CI/CD across multiple application repos. Unlike per-repo workflow configurations, these centralized reusable workflows auto-detect languages (Node.js, Go, Python), apply branch-based semantic versioning (develop=SNAPSHOT, staging=RC, main=release), and complete the deployment loop by automatically updating k8s/global.yaml with new image tags.
