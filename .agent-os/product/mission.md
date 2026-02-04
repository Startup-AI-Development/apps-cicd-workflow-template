# Product Mission

## Pitch

Reusable CI/CD Workflows is a GitHub Actions workflow library that helps development teams standardize their build, test, and deployment pipelines by providing language-aware, branch-based automation that pushes versioned container images and automatically updates Kubernetes manifests.

## Users

### Primary Customers

- **Development Teams**: Engineers working on application repositories who need consistent, reliable CI/CD without writing complex workflow configurations
- **Platform/DevOps Engineers**: Teams responsible for maintaining infrastructure and ensuring deployment consistency across multiple applications

### User Personas

**Backend Developer** (25-40 years old)
- **Role:** Software Engineer
- **Context:** Works on microservices in a team using multiple languages (Node.js, Go, Python)
- **Pain Points:** Writing and maintaining CI/CD workflows for each repo, inconsistent build processes across projects, manual image tagging errors
- **Goals:** Focus on application code rather than CI/CD configuration, reliable automated deployments

**Platform Engineer** (28-45 years old)
- **Role:** DevOps/Platform Engineer
- **Context:** Manages infrastructure for multiple application teams, ensures deployment standards
- **Pain Points:** Each team implements CI/CD differently, difficult to enforce standards, tracking which image versions are deployed
- **Goals:** Centralized CI/CD management, consistent versioning strategy, automated manifest updates

## The Problem

### Fragmented CI/CD Configurations

Each application repository maintains its own CI/CD workflow, leading to inconsistency, duplication, and maintenance overhead. Teams spend significant time debugging pipeline issues instead of building features.

**Our Solution:** Provide centralized, reusable workflows that teams reference with a single `uses:` directive.

### Manual Versioning Errors

Teams manually tag container images or use inconsistent naming conventions, making it difficult to track which versions are deployed in each environment.

**Our Solution:** Automatic semantic versioning based on branch (develop=SNAPSHOT, staging=RC, main=release) with consistent tag formats.

### Deployment Manifest Drift

Kubernetes manifests often become out of sync with actual deployed images, requiring manual updates and creating deployment confusion.

**Our Solution:** Automatically update k8s/global.yaml with the new image tag after successful builds.

## Differentiators

### Language Auto-Detection

Unlike generic CI/CD templates that require manual configuration per language, our workflows automatically detect the project language (Node.js, Go, Python) and apply appropriate build/test commands. This results in zero-configuration CI/CD for supported languages.

### Branch-Based Semantic Versioning

Unlike manual tagging or commit-SHA-based versioning, we provide a clear versioning strategy tied to Git branches. Developers immediately know the stability level of any image (SNAPSHOT for development, RC for staging, release for production).

### Integrated Manifest Updates

Unlike workflows that only build and push images, our solution completes the deployment loop by automatically updating Kubernetes manifests. This eliminates the manual step between "image built" and "manifest updated".

## Key Features

### Core Features

- **Language Auto-Detection:** Automatically identifies Node.js, Go, Python, or applies generic build steps based on project files
- **Multi-Stage Build Support:** Optimized Dockerfile builds for each detected language
- **Automated Testing:** Runs language-appropriate test commands before building images
- **Container Registry Push:** Pushes images to ghcr.io with proper authentication and permissions

### Versioning Features

- **Branch-Based Tagging:** Automatic tag generation (develop=SNAPSHOT, staging=RC, main=release)
- **Semantic Version Tags:** Consistent versioning format across all applications
- **Git SHA Tracking:** Includes commit SHA in image metadata for traceability

### Integration Features

- **Manifest Auto-Update:** Automatically updates k8s/global.yaml with new image tags
- **Reusable Workflow Pattern:** App repos call workflows with simple `uses:` syntax
- **Workflow Inputs:** Configurable parameters for customization when needed
