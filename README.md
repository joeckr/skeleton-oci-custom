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

- **Rootless Image Builds**: Engineered to build cleanly in unprivileged, rootless container builders (such as rootless Podman, Buildah, or unprivileged CI runners) without requiring host root or privileged daemon sockets.
- **Custom Image Starter**: Boilerplate `Dockerfile` set up to build custom application images with configurable base images and tags.
- **OpenShift Compliance**: Configured to run without root privileges and handle arbitrary user IDs gracefully.
- **Talos Linux Compatibility**: Follows security best practices suitable for immutable, secure-by-default Kubernetes operating systems.
- **Dual Ingress & Route Support**: Helm chart cleanly toggles between standard Kubernetes `Ingress` (`networking.k8s.io/v1`) for Talos and OpenShift `Route` (`route.openshift.io/v1`).
- **Helm Chart Included**: Comes with a ready-to-use Helm chart (`chart/`) for templated, reproducible deployments.
- **Matrix CI Pipelines**: Uses `joeckr/ci-templates` workflows (`build-oci-custom.yml`, `push-helm-ghcr.yml`, `semantic.yml`) to automatically build, scan, tag, and push images and Helm charts.
- **Version Matrix Configuration**: Control target image versions, base images, and base tags via `versions.json`.
- **Mise Task Automation**: Built-in `mise` tasks for local compose testing, `podman play kube` simulation, linting, rendering, and cluster deployment.
- **Code Quality & Linting**: Pre-configured with `hk` hooks for pre-commit checks (`actionlint`, `zizmor`, `yamllint`, `helm-lint`).

## Repository Structure

- `Dockerfile`: Template Dockerfile demonstrating secure patterns (non-root user, group 0 permissions) for custom builds.
- `chart/`: Accompanying Helm chart for deploying the application to OpenShift or Talos Linux.
- `versions.json`: Build matrix defining target image version, base image, base tag, and release flags.
- `compose.yml`: For local testing and development of the custom image.
- `mise.toml`: Task runner configuration automating build, compose, play, lint, and cluster deployment workflows.
- `.github/workflows/`:
  - `release.yml`: Production release pipeline (semantic versioning, custom OCI build & push, Helm chart push to GHCR).
  - `test_release.yml`: PR validation pipeline running dry-run builds and test releases.
  - `lint.yml`: Validates workflows, Helm charts, and repository code.
  - `security.yml`: Code and dependency security scanning.
- `hk.pkl`: Pre-commit hook configuration managed with `hk`.

## Security & Compliance Architecture

Both OpenShift and Talos Linux prioritize workload security and least privilege, but they enforce and evaluate constraints through different mechanisms. This repository is architected to satisfy both environments without code changes.

### OpenShift Compliance (`restricted-v2` SCC)

OpenShift uses **Security Context Constraints (SCC)** to control pod permissions. Under the default `restricted-v2` SCC:
- **Arbitrary Dynamic UIDs**: OpenShift assigns a random UID from a dedicated per-namespace range (e.g., `1000670000`). Containers cannot assume a fixed UID like `1000`.
- **Root Group (GID 0)**: Files and directories required at runtime must be owned by group 0 (`chgrp -R 0`) with group read/write permissions (`chmod -R g+rwX`) so the dynamically assigned UID can access them.
- **Dropped Capabilities**: Drops standard root capabilities (`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `SYS_CHROOT`, etc.) and permits only unprivileged operations (and `NET_BIND_SERVICE` when needed).
- **Unprivileged Ports**: Containers must listen on non-privileged ports (> 1024), such as port `8080`.
- **Routing**: OpenShift natively supports `route.openshift.io/v1` Routes for external ingress traffic.

### Talos Linux Compliance (Kubernetes PSS `restricted`)

Talos Linux is an immutable, minimal, secure-by-default Kubernetes operating system with no SSH, no interactive shell, and an immutable root filesystem. In Talos clusters:
- **Pod Security Standards (PSS)**: Workload namespaces enforce the Kubernetes **Pod Security Admission (PSA)** `restricted` profile.
- **Must Run As Non-Root**: The pod specification must set `securityContext.runAsNonRoot: true`. Containers cannot execute as UID 0.
- **Drop All Capabilities**: The container specification must explicitly drop all Linux capabilities (`capabilities: drop: ["ALL"]`).
- **Disallow Privilege Escalation**: Must set `securityContext.allowPrivilegeEscalation: false` to prevent child processes from acquiring more privileges than the parent.
- **Seccomp Profile**: Pods must enforce `seccompProfile: { type: RuntimeDefault }`.
- **Credential Protection**: Best practice sets `automountServiceAccountToken: false` to avoid leaking Kubernetes API tokens to application containers unless explicitly needed.
- **Standard Ingress & Storage**: Talos relies on standard Kubernetes `networking.k8s.io/v1` `Ingress` (e.g., via Cilium, Traefik, or Ingress-NGINX) and CSI storage providers (e.g., Local Path Provisioner, OpenEBS Mayastor, Rook-Ceph).

### Rootless Build Environment Compliance

Building container images inside secure or unprivileged environments (such as rootless Podman/Buildah on developer workstations, or unprivileged Kubernetes CI runners like Tekton or Kaniko) requires that the build process itself does not rely on host `root` privileges or the legacy root-owned Docker daemon socket (`/var/run/docker.sock`).

This repository's `Dockerfile` is engineered for complete rootless build support:
- **No Host Root Required**: Builds execute and succeed cleanly under unprivileged user namespaces without needing `sudo` or privileged container builders.
- **User Namespace Friendly Permissions**: Layer modifications rely on `chgrp -R 0` and group-based permissions (`g+rwX`), which map cleanly into subordinate UID/GID allocations (`/etc/subuid` and `/etc/subgid`) without failing on host-restricted `chown` operations.
- **Atomic Copy Permissions**: Uses `COPY --chmod=755` directly rather than invoking privileged `chmod` steps in subsequent `RUN` layers.
- **Unprivileged Local Build**: Run `mise run build` (`podman buildx build --platform linux/amd64 -t ghcr.io/joeckr/oci-custom:test . --load`) or `mise run compose` to build locally without root escalation.

### Compliance Matrix

| Security Dimension | OpenShift (`restricted-v2` SCC) | Talos Linux (Kubernetes PSS `restricted`) | Implementation in This Repo |
|---|---|---|---|
| **Build Execution** | Rootless builder compatible | Rootless builder compatible | Builds unprivileged via rootless Podman/Buildah (`mise run build`) |
| **User ID** | Dynamic arbitrary UID (`MustRunAsRange`) | Non-root UID (`runAsNonRoot: true`) | `USER 1031` in Dockerfile + `runAsNonRoot: true` in Helm |
| **Group Permissions** | Requires GID 0 (`root`) with `g+rwX` | Compatible with GID 0 / unprivileged groups | `chgrp -R 0` & `chmod -R g+rwX` on runtime paths |
| **Capabilities** | Drops root caps; allows `NET_BIND_SERVICE` | Must drop `ALL` capabilities | `capabilities.drop: ["ALL"]` in Helm chart |
| **Privilege Escalation** | Prohibited | `allowPrivilegeEscalation: false` | Configured in Helm `securityContext` |
| **Seccomp Profile** | `RuntimeDefault` | `RuntimeDefault` or `Localhost` | `seccompProfile: { type: RuntimeDefault }` |
| **Service Account Token** | Optional | Recommended disabled | `automountServiceAccountToken: false` |
| **Port Binding** | Unprivileged (> 1024) | Unprivileged (> 1024) | Listens on port `8080` |
| **Ingress Layer** | OpenShift Route (`route.openshift.io/v1`) | Kubernetes Ingress (`networking.k8s.io/v1`) | Configurable via `ingress.route: "true"` or `"false"` |
| **Storage Layer** | OpenShift StorageClass | Talos CSI StorageClass | Standard PVC template with configurable `storageClass` |

---

## Local Environment & Podman Setup

To ensure containerized applications and Helm charts tested locally run cleanly when deployed to OpenShift or Talos Linux, this repository is designed to be used alongside the Podman configuration in [joeckr/dotfiles](https://github.com/joeckr/dotfiles).

The dotfiles repository provides a centralized [`containers.conf`](https://github.com/joeckr/dotfiles/blob/main/containers/containers.conf) (deployed to `~/.config/containers/containers.conf`) that configures Podman to simulate OpenShift and Talos Linux runtime restrictions:

| Security Rule | Podman Configuration | Description |
|---|---|---|
| **Random UID (`MustRunAsRange`)** | `userns = "auto"` | Allocates dynamic subordinate UID/GID ranges from `/etc/subuid` and `/etc/subgid`. Containers run unprivileged without mapping host root. |
| **Drop Capabilities** | `default_capabilities = ["NET_BIND_SERVICE"]` | Drops standard root capabilities (`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `SYS_CHROOT`, etc.) and permits only `NET_BIND_SERVICE`. |
| **Disallow Privileged** | `privileged = false` | Disallows privileged container execution by default. |
| **Seccomp Profile** | `seccomp_profile = "/usr/share/containers/seccomp.json"` | Enforces the runtime default seccomp profile (`RuntimeDefault`). |
| **Namespace Isolation** | `cgroupns`, `ipcns`, `pidns`, `utsns = "private"` | Enforces private container namespaces (host namespaces are forbidden in restricted profiles). |

### macOS Podman Machine Integration

On macOS, the dotfiles installer script (`brew/podman.sh`) automates the machine lifecycle:

1. Deploys `containers/containers.conf` to `~/.config/containers/containers.conf` on the host.
2. Initializing `podman machine init` automatically mounts `~/.config/containers` into `/etc/containers` inside the Fedora CoreOS VM.
3. Automatically symlinks `/etc/containers/containers.conf` to the VM user's config (`~core/.config/containers/containers.conf`) and restarts the Podman API service so all container executions immediately enforce these constraints.

---

## Testing & Validation Process

This repository defines a 3-tier testing process to validate container security, manifest generation, and runtime compatibility from local development through to production cluster deployment.

```
┌─────────────────────────┐     ┌─────────────────────────┐     ┌─────────────────────────┐
│ Tier 1: Custom Test     │ ──> │ Tier 2: Podman Play     │ ──> │ Tier 3: Talos Cluster   │
│ Verify non-root & build │     │ Validate K8s manifests  │     │ Live Helm verification  │
│ (compose.yml)           │     │ (podman play kube)      │     │ (helm install)          │
└─────────────────────────┘     └─────────────────────────┘     └─────────────────────────┘
```

### Tier 1: Custom Image Local Validation (`compose.yml`)

The [`compose.yml`](compose.yml) configuration builds and runs the customized `Dockerfile` containing the adaptations required for OpenShift and Talos Linux:

```sh
# Build and start the compliant container
mise run compose
# or: podman compose up -d --build

# View container logs
mise run logs
# or: podman compose logs -f

# Verify connectivity
curl http://localhost:8080

# Stop compose stack
mise run down
# or: podman compose down
```

**What this verifies:**
- Rootless image build and layer assembly without host root privileges.
- Non-root user execution (`USER 1031`).
- Root group ownership (`chgrp -R 0`) and group read/write permissions (`chmod -R g+rwX`) on runtime directories.
- Unprivileged port binding (`8080`).
- The `entrypoint.sh` script dynamically handling runtime configuration and templating under non-root UIDs.

---

### Tier 2: Local Kubernetes Manifest Testing (`mise run play`)

Before deploying to an actual Kubernetes cluster, you can test the rendered Kubernetes manifests locally using Podman's built-in `play kube` feature.

```sh
# Render templates and play Kubernetes manifests locally
mise run play

# Teardown the played pod and resources
mise run downplay
```

**How `mise run play` works:**
1. Triggers the dependent task `mise run helm-template`, which executes:
   ```sh
   helm dependency build chart/
   helm template test chart/ > rendered.yaml
   ```
2. Executes `podman play kube rendered.yaml`, which:
   - Reads the multi-document Kubernetes YAML (`ConfigMap`, `PersistentVolumeClaim`, `Service`, `Deployment`, `Ingress`).
   - Creates a local Podman pod matching the Kubernetes `Deployment` specification.
   - Applies the pod's `securityContext` (`runAsNonRoot: true`, capabilities drop, seccomp profile).
   - Mounts the `ConfigMap` (`template.txt`) and volume into the container at the specified paths.
   - Exposes container port `8080`.

**Inspecting the local play deployment:**
```sh
# View running pods created by play kube
podman pod ps

# View container status within the pod
podman ps --filter "pod=oci-custom"

# Verify the service responds
curl http://localhost:8080

# Check container logs within the pod
podman logs -f oci-custom-pod-oci-custom
```

**Teardown:**
```sh
mise run downplay
# or: podman play kube rendered.yaml --down
```

---

### Tier 3: Cluster Deployment & Testing on Talos Linux (`mise run helm-install`)

The final phase validates the workload on a live **Talos Linux** Kubernetes cluster. This tests real-world Pod Security Admission (PSA) enforcement, CSI storage provisioning, network policies, and Ingress routing.

#### 1. Cluster Prerequisites & Configuration

Ensure your `kubectl` context points to your Talos cluster:
```sh
kubectl config current-context
# Example: admin@my-talos-cluster
```

Ensure the container image is accessible to your Talos nodes (e.g., built and pushed to GitHub Container Registry `ghcr.io` or your local registry):
```sh
# Build image locally with target tag
mise run build
```

Configure `chart/values.yaml` for Talos Linux:
- **Ingress vs. Route**: Ensure `ingress.route` is set to `"false"` (default) so Helm generates standard Kubernetes `networking.k8s.io/v1` `Ingress` rather than an OpenShift Route:
  ```yaml
  ingress:
    name: template-ingress
    host: "test.yourdomain.com"
    route: "false"                 # "false" for Talos / vanilla Kubernetes; "true" for OpenShift
    className: "nginx"             # e.g., "nginx", "traefik", or "cilium"
    path: /
    pathType: "Prefix"
  ```
- **StorageClass**: If your Talos cluster uses a specific CSI storage provisioner (e.g., `local-path`, `mayastor`, `ceph-block`), configure `pvc.storageClass` in `values.yaml` or leave it empty `""` to use the cluster's default StorageClass.

#### 2. Linting & Template Validation

```sh
# Lint the chart for syntax and formatting errors
mise run helm-lint

# Inspect the rendered manifests before installation
mise run helm-template
cat rendered.yaml
```

#### 3. Deploying to the Talos Cluster

Install the Helm chart release:
```sh
mise run helm-install
# or: helm install test chart/
```

#### 4. Verifying Talos PSS Compliance & Health

Check the pod status and verify that Talos Linux Pod Security Admission (PSA) allowed the pod to run:

```sh
# Check pod deployment status
kubectl get pods -l app=oci-custom

# Inspect pod details and events for security policy rejections
kubectl describe pod -l app=oci-custom
```

> [!TIP]
> If your namespace enforces the `restricted` Pod Security Standard and there are non-compliant settings (such as missing `runAsNonRoot` or un-dropped capabilities), `kubectl describe pod` will show warning events from the `pod-security` admission controller.

Check the application logs:
```sh
kubectl logs -l app=oci-custom -f
```

Verify that the `ConfigMap` file and persistent storage mounted properly inside the pod:
```sh
kubectl exec -it deployment/oci-custom -- cat /app/template.txt
kubectl exec -it deployment/oci-custom -- ls -la /tmp/data
```

Verify network access via port-forwarding:
```sh
kubectl port-forward svc/template-service 8080:8080
# In another terminal:
curl http://localhost:8080
```

#### 5. Uninstalling from the Talos Cluster

When testing is complete, clean up the release:
```sh
mise run helm-uninstall
# or: helm uninstall test
```

---

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
Navigate to the `chart/` directory and update `Chart.yaml`, `values.yaml`, and the templates to reflect your application's specifics. Choose between Ingress (`ingress.route: "false"`) for Talos/vanilla Kubernetes or Route (`ingress.route: "true"`) for OpenShift.

### 6. Run the Test Suite
Validate changes through the 3-tier process: `mise run compose` (local container validation), `mise run play` (manifest test), and `mise run helm-install` (Talos cluster test).

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
| `mise run helm-dep` | `helm dependency build chart/` | Build Helm chart dependencies. |
| `mise run helm-lint` | `helm lint chart/` | Lint the Helm chart. |
| `mise run helm-template` | `helm template test chart/ > rendered.yaml` | Render Helm chart templates to `rendered.yaml`. |
| `mise run helm-install` | `helm install test chart/` | Install the Helm chart to the current Kubernetes cluster. |
| `mise run helm-uninstall` | `helm uninstall test` | Uninstall the Helm chart release from the cluster. |
| `mise run build` | `podman buildx build --platform linux/amd64 -t ghcr.io/joeckr/oci-custom:test . --load` | Build local test container image for `linux/amd64`. |
| `mise run trivy-fs` | `trivy fs .` | Scan local repository filesystem for security vulnerabilities. |
| `mise run trivy-image` | `trivy image ghcr.io/joeckr/oci-custom:test` | Build image and run Trivy vulnerability scan on container. |

Checks run by `hk` include `hadolint`, `yamllint`, `actionlint`, `tombi`, `betterleaks`, and `shellcheck`.


## Support

If you find this project useful, consider supporting my work on [Ko-fi](https://ko-fi.com/joeckr):

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/joeckr)

## License

Please refer to the `LICENSE` file for details.
