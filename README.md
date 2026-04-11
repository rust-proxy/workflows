# Reusable Workflows for Rust Projects

This repository contains a set of reusable GitHub Actions workflows designed for building, releasing, and publishing Rust projects.

## Workflows

1.  [Build Rust Binary (`rust-build.yml`)](#1-build-rust-binary)
2.  [Publish Release (`release-publish.yml`)](#2-publish-release)
3.  [Build and Publish Docker Image (`docker-publish.yml`)](#3-build-and-publish-docker-image)

---

### 1. Build Rust Binary

**Path**: `.github/workflows/rust-build.yml`

This workflow compiles Rust binaries for multiple targets, runs checks, lints, and tests, and uploads the compiled binaries as artifacts. The build matrix is dynamically generated from a TOML configuration file.

#### Usage Example

```yaml
jobs:
  build:
    uses: rust-proxy/workflows/.github/workflows/rust-build.yml@main
    with:
      packages: 'clash-rs, clash-bin'
      target-config-file: '.github/targets.toml'
```

#### Inputs

| Name                      | Description                                                                                             | Type    | Required | Default              |
| ------------------------- | ------------------------------------------------------------------------------------------------------- | ------- | -------- | -------------------- |
| `rust-toolchain`          | Default Rust toolchain to use (can be overridden by the target matrix).                                 | string  | No       | `stable`             |
| `packages`                | Comma-separated list of package names to build.                                                         | string  | Yes      |                      |
| `target-config-file`      | Path to the TOML file containing the build matrix targets.                                              | string  | No       | `.github/target.toml`|
| `run-tests`               | A global flag to enable or disable tests. Can be overridden by `skip-test` in the matrix.               | boolean | No       | `true`               |
| `cache-key-prefix`        | A prefix for the cache key to prevent collisions if using this workflow multiple times in one job.      | string  | No       | `rust-build`         |
| `rustflags`               | Additional RUSTFLAGS to pass to the compiler for all targets.                                           | string  | No       | `''`                 |
| `enable-tmate`            | If `true`, starts a tmate session on failure for live debugging.                                        | boolean | No       | `false`              |
| `only-clippy-tests-on-pr` | If `true`, runs clippy and tests only on `pull_request` events. If `false`, they run on any event.       | boolean | No       | `false`              |

---

### 2. Publish Release

**Path**: `.github/workflows/release-publish.yml`

This workflow creates GitHub releases. It handles both versioned releases (from `v*` tags) and rolling "latest" pre-releases (from pushes to the main branch). For versioned releases, it automatically generates a changelog using `git-cliff`.

#### Usage Example

```yaml
jobs:
  release:
    needs: build
    uses: rust-proxy/workflows/.github/workflows/release-publish.yml@main
    with:
      create-latest-on-push: ${{ github.ref == 'refs/heads/main' }}
    secrets: inherit
```

#### Inputs

| Name                    | Description                                                                    | Type    | Required | Default           |
| ----------------------- | ------------------------------------------------------------------------------ | ------- | -------- | ----------------- |
| `cliff-config`          | Path to the `git-cliff` configuration file for changelog generation.           | string  | No       | `.github/cliff.toml` |
| `create-latest-on-push` | If `true`, it will create or update a `latest` pre-release.                    | boolean | No       | `false`           |
| `publish-mode`          | Set to `draft` to create draft releases instead of publishing them immediately. | string  | No       | `publish`         |

---

### 3. Build and Publish Docker Image

**Path**: `.github/workflows/docker-publish.yml`

This workflow builds a multi-platform Docker image, tags it, and pushes it to one or more container registries. GitHub Container Registry (GHCR) is included by default and can be disabled. Additional registries (Docker Hub, Quay, etc.) are configured via the `registrys` input.

#### Usage Example

```yaml
jobs:
  docker:
    needs: build
    if: startsWith(github.ref, 'refs/tags/v')
    uses: rust-proxy/workflows/.github/workflows/docker-publish.yml@main
    with:
      image-name: 'clash-rs'
      server-binary-amd64: 'clash-rs-x86_64-unknown-linux-musl'
      server-binary-arm64: 'clash-rs-aarch64-unknown-linux-musl'
    secrets:
      registrys: |
        myuser:${{ secrets.DOCKERHUB_TOKEN }}@docker.io
        myuser:${{ secrets.QUAY_TOKEN }}@quay.io
```

> **GHCR** is always logged in automatically via `GITHUB_TOKEN`. Set `disable-ghcr: true` to opt out.

#### Inputs

| Name                  | Description                                                        | Type    | Required | Default                     |
| --------------------- | ------------------------------------------------------------------ | ------- | -------- | --------------------------- |
| `disable-ghcr`        | Set to `true` to skip GHCR login and exclude GHCR tags             | boolean | No       | `false`                     |
| `docker-file`         | Path to the `Dockerfile`.                                          | string  | No       | `.github/Dockerfile`        |
| `image-name`          | The base name for the Docker image.                                | string  | No       | `bin`                       |
| `platforms`           | A comma-separated list of platforms to build for.                  | string  | No       | `linux/amd64,linux/arm64`   |
| `binary-dir`          | The directory where downloaded binary artifacts are stored.        | string  | No       | `./docker-bins`             |
| `server-binary-amd64` | The filename of the amd64 binary.                                  | string  | No       | `bin-x86_64-linux-musl`     |
| `server-binary-arm64` | The filename of the arm64 binary.                                  | string  | No       | `bin-aarch64-linux-musl`    |

#### Secrets

| Name        | Description                                                                      | Required |
| ----------- | -------------------------------------------------------------------------------- | -------- |
| `registrys` | Newline-separated list of registries: `username:password@registry`               | No       |
