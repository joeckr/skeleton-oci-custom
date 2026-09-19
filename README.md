# Skeleton OCI Custom

This repository serves as a starter template for building, packaging, and deploying your own custom container images. It provides secure-by-default container configurations compliant with **OpenShift** (arbitrary UID support) and **Talos Linux**, automated multi-architecture CI/CD workflows, and an accompanying production-ready **Helm chart**.

## Purpose

Building custom container images for modern, security-hardened Kubernetes environments like OpenShift (enforcing strict Security Context Constraints) and Talos Linux requires adhering to strict security defaults from the start. Images must run without root privileges, avoid privilege escalation, and accommodate arbitrary user IDs.

This skeleton provides an out-of-the-box foundation to:
- **Build Custom Images**: Jumpstart development of your own containerized services and applications.
- **Enterprise Security Defaults**: Run as non-root (`USER 1031`), support OpenShift arbitrary UIDs (`chgrp -R 0` and `chmod -R g+rwX`), and satisfy restricted pod security standards.
- **Automated CI/CD**: Build multi-platform images (amd64, arm64), run vulnerability scans (Trivy), generate Software Bills of Materials (SBOM), and publish to GitHub Container Registry (GHCR) using reusable CI templates.
- **Deploy with Helm**: Deploy seamlessly with a ready-to-use Helm chart (`chart/`) supporting Kubernetes Deployments, Services, Ingress, OpenShift Routes, Persistent Volume Claims (PVC), and ConfigMaps.
- **Local Development**: Rapidly iterate and verify builds locally using Podman Compose.

## Features

- **Custom Image Starter**: Boilerplate `Dockerfile` set up to build custom application images with configurable base images and tags.
- **OpenShift Compliance**: Configured to run without root privileges and handle arbitrary user IDs gracefully.
- **Talos Linux Compatibility**: Follows security best practices suitable for immutable, secure-by-default Kubernetes operating systems.
- **Helm Chart Included**: Comes with a ready-to-use Helm chart (`chart/`) for templated, reproducible deployments.
- **Matrix CI Pipelines**: Uses `joeckr/ci-templates` workflows (`build-oci-custom.yml`, `push-helm-ghcr.yml`, `semantic.yml`) to automatically build, scan, tag, and push images and Helm charts.
- **Version Matrix Configuration**: Control target image versions, base images, and base tags via `versions.json`.
- **Local Testing**: Includes `compose.yml` for local container verification and local Helm testing with Podman Play Kube.
- **Code Quality & Linting**: Pre-configured with `hk` hooks for pre-commit checks (`actionlint`, `zizmor`, `yamllint`, `helm-lint`).

## Repository Structure

- `Dockerfile`: Template Dockerfile demonstrating secure patterns (non-root user, group 0 permissions) for custom builds.
- `chart/`: Accompanying Helm chart for deploying the application to Kubernetes or OpenShift.
- `versions.json`: Build matrix defining target image version, base image, base tag, and release flags.
- `compose.yml`: For local testing and development.
- `mise.toml`: Local tool definitions and task runner (`mise run compose`, `mise run build`, etc.).
- `.github/workflows/`:
  - `release.yml`: Production release pipeline (semantic versioning, custom OCI build & push, Helm chart push to GHCR).
  - `test_release.yml`: PR validation pipeline running dry-run builds and test releases.
  - `lint.yml`: Validates workflows, Helm charts, and repository code.
  - `security.yml`: Code and dependency security scanning.
- `hk.pkl`: Pre-commit hook configuration managed with `hk`.

## Local Environment & Podman Setup

To ensure containerized applications and Helm charts tested locally run cleanly when deployed to OpenShift or Talos Linux, this repository is designed to be used alongside the Podman configuration in [joeckr/dotfiles](https://github.com/joeckr/dotfiles).

The dotfiles repository provides a centralized [`containers.conf`](https://github.com/joeckr/dotfiles/blob/main/containers/containers.conf) (deployed to `~/.config/containers/containers.conf`) that configures Podman to simulate OpenShift's default **`restricted-v2` Security Context Constraints (SCC)**:

| OpenShift SCC Rule | Podman Configuration | Description |
|---|---|---|
| **Random UID (`MustRunAsRange`)** | `userns = "auto"` | Allocates dynamic subordinate UID/GID ranges from `/etc/subuid` and `/etc/subgid`. Containers run unprivileged without mapping host root. |
| **Drop Capabilities** | `default_capabilities = ["NET_BIND_SERVICE"]` | Drops standard root capabilities (`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `SYS_CHROOT`, etc.) and permits only `NET_BIND_SERVICE`. |
| **Disallow Privileged** | `privileged = false` | Disallows privileged container execution by default. |
| **Seccomp Profile** | `seccomp_profile = "/usr/share/containers/seccomp.json"` | Enforces the runtime default seccomp profile (`RuntimeDefault`). |
| **Namespace Isolation** | `cgroupns`, `ipcns`, `pidns`, `utsns = "private"` | Enforces private container namespaces (host namespaces are forbidden in restricted SCC). |

### macOS Podman Machine Integration

On macOS, the dotfiles installer script (`brew/podman.sh`) automates the machine lifecycle:

1. Deploys `containers/containers.conf` to `~/.config/containers/containers.conf` on the host.
2. Initializing `podman machine init` automatically mounts `~/.config/containers` into `/etc/containers` inside the Fedora CoreOS VM.
3. Automatically symlinks `/etc/containers/containers.conf` to the VM user's config (`~core/.config/containers/containers.conf`) and restarts the Podman API service so all container executions immediately enforce these constraints.

## Testing with Podman Compose

The [`compose.yml`](compose.yml) file builds and runs the container with the security adaptations required for OpenShift and Talos Linux:

```sh
# Build and start the compliant container
podman compose up -d --build

# Or via mise
mise run compose
```

This verified configuration applies:
- Non-root user execution (`USER 1031`).
- Root group ownership (`chgrp -R 0`) and group read/write permissions (`chmod -R g+rwX`) on runtime directories.
- Unprivileged port bindings (`8080`).

### Stopping Containers & Viewing Logs

```sh
# Stop compose stack
podman compose down
# or: mise run down

# View logs
podman compose logs -f
# or: mise run logs
```

### Local Helm Testing (Podman Play Kube)

Test rendered Helm chart manifests directly in Podman without requiring a remote cluster:

```sh
# Render Helm template and run pods locally via podman play kube
mise run play

# Stop and tear down local pods
mise run downplay
```

## Getting Started

### 1. Use as a Template
Clone or use this repository as a template for your own custom image project:
```bash
git clone https://github.com/joeckr/skeleton-oci.git my-custom-image
cd my-custom-image
```

### 2. Customize the Dockerfile
Update `Dockerfile` with your application dependencies, build stages, or assets. The template is pre-configured with base image arguments:
```dockerfile
ARG IMAGE=nginx
ARG TAG=1.31.5-alpine
ARG REGISTRY=docker.io/library
FROM $REGISTRY/$IMAGE

ARG VERSION=1.31.5
...
```

Ensure any writable directories maintain non-root and OpenShift arbitrary UID compatibility:
```dockerfile
RUN chgrp -R 0 /app && \
    chmod -R g+rwX /app

USER 1031
```

### 3. Configure the Build Matrix
Define the versions you want to build in `versions.json`:
```json
[
  {
    "version": "1.31.5",
    "base-image": "nginx",
    "base-tag": "1.31.5-alpine",
    "latest": true,
    "lts": false
  }
]
```
The CI workflow passes `VERSION` (`matrix.version.version`), `IMAGE` (`matrix.version.base-image`), and `TAG` (`matrix.version.base-tag`) directly into the Docker build as build arguments.

### 4. Configure CI Workflows
In `.github/workflows/release.yml` and `.github/workflows/test_release.yml`, update the `image` input to match your target image name:
```yaml
uses: joeckr/ci-templates/.github/workflows/build-oci-custom.yml@...
with:
  image: "my-custom-image"
```

### 5. Update the Helm Chart
Navigate to the `chart/` directory and update `Chart.yaml` and `values.yaml` to reflect your application name, container image path, port configurations, and ingress settings.

### 6. Test Locally

Build and run the container locally with Podman Compose:

```bash
podman compose up --build
# or: mise run compose
```

Or test rendered Helm chart manifests directly via Podman Play Kube:

```bash
mise run play
```

## Code Quality & Hooks

This project uses [`mise`](https://mise.jdx.dev/) for task execution and [`hk`](https://github.com/jdx/hk) for pre-commit quality enforcement:

```bash
# Setup tools and git hooks
mise run install

# Run all linters and hook checks
mise run hk # or mise run check
```

### Available mise Tasks

The following tasks are defined in [`mise.toml`](mise.toml):

| Task | Command | Description |
|---|---|---|
| `mise run install` | `hk install --mise` | Install Git hooks (`pre-commit` and `commit-msg`). |
| `mise run hk` *(or `check`)* | `hk check --all` | Run all checks across the repository. |
| `mise run compose` | `podman compose up -d --build` | Start local container environment with Podman Compose. |
| `mise run down` | `podman compose down` | Stop local Podman Compose stack. |
| `mise run logs` | `podman compose logs -f` | Follow Podman Compose logs. |
| `mise run play` | `podman play kube rendered.yaml` | Test Helm chart manifests locally with Podman Play Kube. |
| `mise run downplay` | `podman play kube rendered.yaml --down` | Stop and tear down Podman Play Kube pods. |
| `mise run helm-lint` | `helm lint chart/` | Lint the Helm chart. |
| `mise run helm-template` | `helm template test chart/ > rendered.yaml` | Render Helm chart templates to `rendered.yaml`. |
| `mise run helm-dep` | `helm dependency build chart/` | Build Helm chart dependencies. |
| `mise run build` | `podman buildx build --platform linux/amd64 -t ghcr.io/joeckr/oci-modified:test . --load` | Build local test container image for `linux/amd64`. |
| `mise run trivy-fs` | `trivy fs .` | Scan local repository filesystem for security vulnerabilities. |
| `mise run trivy-image` | `trivy image ghcr.io/joeckr/oci-modified:test` | Build image and run Trivy vulnerability scan on container. |

Checks run by `hk` include `hadolint`, `yamllint`, `actionlint`, `tombi`, `betterleaks`, and `shellcheck`.


## Support

If you find this project useful, consider supporting my work on [Ko-fi](https://ko-fi.com/joeckr):

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/joeckr)

## License

Please refer to the `LICENSE` file for details.
