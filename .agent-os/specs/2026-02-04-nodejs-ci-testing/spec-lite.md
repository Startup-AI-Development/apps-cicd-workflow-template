# Spec Summary (Lite)

Refactor Node.js CI workflow to run build first, then execute unit tests, integration tests, and security audit in parallel. All three jobs must pass before Docker image is built. Test jobs gracefully succeed if no tests exist in expected folders.
