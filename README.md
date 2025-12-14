# Shared Workflows

Reusable GitHub Actions workflows for the P7 Strato platform.

## Available Workflows

| Workflow | Description |
|----------|-------------|
| `build.yml` | Language-specific build validation (Python, .NET, Node.js) |
| `pr-check.yml` | PR build verification with optional tests |
| `containerize.yml` | Build and push multi-arch Docker images |
| `manifest-update.yml` | Update GitOps K8s manifests |
| `service.yml` | All-in-one pipeline (build + containerize + manifest update) |

## Quick Start

### Option 1: All-in-One Pipeline

For most services, use `service.yml` for a complete CI/CD pipeline:

```yaml
name: Prod Build

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: P7-Chaos-Academy/shared-workflows/.github/workflows/service.yml@main
    with:
      language: python  # or 'dotnet' or 'node'
      image_name: my-service
      gitops_repo: P7-Chaos-Academy/gitops
      manifest_path: my-service/deployment.yaml
      image_pattern: 'image: cgamel/my-service:.*'
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      GITOPS_REPO_TOKEN: ${{ secrets.GITOPS_REPO_TOKEN }}
```

### Option 2: Modular Workflows

For more control, chain individual workflows:

```yaml
name: Prod Build

on:
  push:
    branches: [main]

jobs:
  build:
    uses: P7-Chaos-Academy/shared-workflows/.github/workflows/build.yml@main
    with:
      language: dotnet

  containerize:
    needs: build
    uses: P7-Chaos-Academy/shared-workflows/.github/workflows/containerize.yml@main
    with:
      image_name: strato-api
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

  update-gitops:
    needs: containerize
    uses: P7-Chaos-Academy/shared-workflows/.github/workflows/manifest-update.yml@main
    with:
      gitops_repo: P7-Chaos-Academy/gitops
      manifest_path: strato-api/deployment.yaml
      image_pattern: 'image: cgamel/strato-api:.*'
      new_image: ${{ needs.containerize.outputs.full_image }}
    secrets:
      GITOPS_REPO_TOKEN: ${{ secrets.GITOPS_REPO_TOKEN }}
```

## Workflow Reference

### build.yml

Validates that the application compiles/builds successfully.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `language` | Yes | - | `python`, `dotnet`, or `node` |
| `python_version` | No | `3.11` | Python version |
| `dotnet_version` | No | `8.0.x` | .NET SDK version |
| `node_version` | No | `20` | Node.js version |
| `working_directory` | No | `.` | Build directory |
| `build_command` | No | - | Custom build command |
| `install_command` | No | - | Custom install command |

### pr-check.yml

Same as `build.yml` but includes optional test execution.

| Additional Input | Required | Default | Description |
|------------------|----------|---------|-------------|
| `run_tests` | No | `true` | Run tests after build |
| `test_command` | No | - | Custom test command |

### containerize.yml

Builds and pushes Docker images with multi-arch support.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image_name` | Yes | - | Image name (without registry prefix) |
| `dockerfile` | No | `./Dockerfile` | Dockerfile path |
| `context` | No | `.` | Build context |
| `platforms` | No | `linux/amd64,linux/arm64` | Target platforms |
| `push` | No | `true` | Push to registry |
| `tag_latest` | No | `false` | Also tag as `latest` |
| `build_args` | No | - | Build arguments |
| `cache_enabled` | No | `true` | Enable layer caching |

| Secret | Required | Description |
|--------|----------|-------------|
| `DOCKER_USERNAME` | Yes | Docker Hub username |
| `DOCKER_PASSWORD` | Yes | Docker Hub token |

| Output | Description |
|--------|-------------|
| `image_tag` | Short SHA tag (e.g., `a1b2c3d4`) |
| `full_image` | Full reference (e.g., `user/image:a1b2c3d4`) |

### manifest-update.yml

Updates image references in GitOps K8s manifests.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `gitops_repo` | Yes | - | GitOps repo (`org/repo`) |
| `manifest_path` | Yes | - | Path to manifest file |
| `image_pattern` | Yes | - | Regex to match image line |
| `new_image` | Yes | - | New image reference |
| `commit_message` | No | Auto | Custom commit message |

| Secret | Required | Description |
|--------|----------|-------------|
| `GITOPS_REPO_TOKEN` | Yes | PAT with repo write access |

### service.yml

All-in-one pipeline combining build, containerize, and manifest update.

Accepts all inputs from individual workflows

## Required Secrets

Configure these in your repository settings:

| Secret | Description |
|--------|-------------|
| `DOCKER_USERNAME` | Docker Hub username |
| `DOCKER_PASSWORD` | Docker Hub access token |
| `GITOPS_REPO_TOKEN` | GitHub PAT with write access to gitops repo |
