# CI/CD Pipeline Documentation

This document describes the CI/CD pipeline implemented for the MobileEtnoV2_backend project.

## Overview

The project uses GitHub Actions for continuous integration and delivery.

### 1. Continuous Integration (CI)
- **Workflow**: [rust.yml](file:///c:/Users/ander/Desktop/MobileEtnoV2_backend/.github/workflows/rust.yml)
- **Triggers**: Pushes and Pull Requests to `main` and `develop`.
- **Steps**:
  - Code formatting check (`cargo fmt`).
  - Linting (`cargo clippy`).
  - Build and unit testing (`cargo test`).
  - Includes a live Postgres service for integration tests.

### 2. Continuous Delivery (CD)
- **Workflow**: [docker.yml](file:///c:/Users/ander/Desktop/MobileEtnoV2_backend/.github/workflows/docker.yml)
- **Triggers**: Pushes to `main` and manual dispatch.
- **Outcome**: Builds a production-ready Docker image and pushes it to GitHub Container Registry (GHCR).
- **Image URL**: `ghcr.io/<github-username>/mobileetno-backend`

---

## Technical Notes & Fixes

During the implementation, the following critical issues were resolved to ensure a stable pipeline:

### 💡 Committing [Cargo.lock](file:///c:/Users/ander/Desktop/MobileEtnoV2_backend/Cargo.lock)
Initially, [Cargo.lock](file:///c:/Users/ander/Desktop/MobileEtnoV2_backend/Cargo.lock) was ignored. For standalone applications (bins), ensuring [Cargo.lock](file:///c:/Users/ander/Desktop/MobileEtnoV2_backend/Cargo.lock) is version-controlled is mandatory for Docker builds to ensure the environment exactly matches your local state and to prevent "file not found" errors during the `COPY` stage.

### 🦀 Rust Compiler Version
The builder image was upgraded to **Rust 1.88**. Modern crates (like `axum`, `time`, and `home`) have moved their Minimum Supported Rust Version (MSRV) beyond 1.85. The [Dockerfile](file:///c:/Users/ander/Desktop/MobileEtnoV2_backend/Dockerfile) now reflects this requirement.

---

## How to use the CD Pipeline

### Manual Trigger
1. Go to the **Actions** tab in your GitHub repository.
2. Select **CD — Build & Push Docker Image**.
3. Click **Run workflow** and select the `main` branch.

### Pulling the Image
Once the pipeline finishes, you can pull the latest image locally:
```bash
docker pull ghcr.io/<your-username>/mobileetno-backend:latest
```
