# The Python Deployment Stack

How a commit becomes a running service. Your locked dependencies and code go in at the top, get layered onto a base image by a Dockerfile, built, scanned and signed, pushed to a registry, and pulled down by something that runs and scales it, all on infrastructure you declared in code. Watch the packet: it starts as code and gets boxed into a container on the way down.

Part of [the Sprawl](../README.md). State as of October 2026. For the styled, interactive version, open [python_deploy_stack.html](../html/python_deploy_stack.html) in a browser.

**Status key:** **PICK** = modern default · alt = solid alternative · ⚠️ edge = maintained, but teetering, too new, or overtaken · ◆×N = appears in N places

## Default builds

- **Single service, minimal ops:** uv.lock → python:slim → multi-stage Dockerfile → buildx → Trivy → GHCR → Cloud Run → GitHub Actions
- **Team on AWS:** uv.lock → python:slim → multi-stage Dockerfile → buildx → Trivy + cosign → ECR → ECS Fargate → Terraform
- **Platform team:** Chainguard base → BuildKit → Syft SBOM + cosign → Harbor or ECR → EKS / GKE → Helm → Gateway API → Argo CD

---

## L9 · Lockfile & artifacts

What goes into the image. A reproducible build starts with exact, locked dependency versions and a clear list of what to leave out.

**Pick:** uv.lock, installed with uv sync --frozen

| Tool | Status | Notes |
|---|---|---|
| uv.lock | **PICK** | Exact versions and hashes for every dependency. --frozen guarantees the image gets precisely what you tested. |
| Your package as a wheel | alt | Build your project into a wheel in one stage and install it in the next. Cleanest for libraries and larger apps. |
| requirements.txt (exported) | alt | uv export --format requirements-txt, for platforms or tools that only understand pip. |
| .dockerignore | **PICK** | Keeps .git, .venv, caches and local secrets out of the build context. Smaller, faster and safer builds. |

---

## L8 · Base image

The operating system and Python interpreter your app sits on. It decides your image size, your security patch load, and whether prebuilt wheels work.

**Pick:** python:3.x-slim (Debian, glibc)

| Tool | Status | Notes |
|---|---|---|
| python:slim | **PICK** | Official Debian-based image with the essentials. glibc means every manylinux wheel installs without compiling. |
| Astral's uv images | alt | Python plus uv preinstalled on Debian slim or Alpine. Handy for the build stage. |
| Chainguard / Wolfi | alt | Minimal images rebuilt daily with near-zero known CVEs. Free tier offers only the latest tags. |
| distroless | alt | No shell, no package manager, tiny attack surface. Its Python versions lag, and debugging is harder. |
| NVIDIA CUDA images | alt | For GPU workloads. Huge; pair with multi-stage builds. |
| python:alpine | ⚠️ edge | Small, but musl libc can't use most prebuilt wheels, so numpy and friends compile from source. Usually slower and bigger in the end. |

---

## L7 · Dockerfile patterns

The recipe. Small differences in ordering and structure decide whether a rebuild takes five seconds or five minutes, and whether the image ships with compilers in it.

**Pick:** Multi-stage build, uv with cache mounts, non-root user

| Tool | Status | Notes |
|---|---|---|
| Multi-stage build | **PICK** | Build stage has compilers and uv; the runtime stage copies only the finished virtualenv. Smaller and safer. |
| uv sync with cache mounts | **PICK** | RUN --mount=type=cache keeps uv's download cache between builds without bloating the image. |
| Lockfile before source | **PICK** | COPY pyproject.toml and uv.lock, install, then COPY the code. The dependency layer stays cached. |
| Non-root USER | **PICK** | Run as an unprivileged user. Limits the damage if the app is compromised. |
| Exec-form CMD | **PICK** | CMD ["uvicorn", ...] so the process gets shutdown signals directly and stops cleanly. |
| Python env vars | alt | PYTHONUNBUFFERED=1 for immediate logs; UV_COMPILE_BYTECODE=1 for faster startup. |

---

## L6 · Build tooling

The engine that executes the Dockerfile. Modern builders cache aggressively, build in parallel, and produce images for several CPU architectures at once.

**Pick:** Docker BuildKit via docker buildx

| Tool | Status | Notes |
|---|---|---|
| BuildKit / buildx | **PICK** | The default builder. Cache mounts, parallel stages, and multi-arch images (amd64 plus arm64) in one command. |
| Podman / Buildah | alt | Daemonless, rootless builds. Standard on Red Hat systems. |
| Depot | alt | Managed remote builders with persistent cache. Much faster CI builds for a fee. |
| Cloud Native Buildpacks | alt | Build an image from source with no Dockerfile. Cloud Run and Heroku use them; less control. |
| kaniko (Chainguard fork) | ⚠️ edge | Builds images inside Kubernetes without a Docker daemon. Google archived it in June 2025; Chainguard keeps a maintenance-only fork. |

---

## L5 · Scan, sign & SBOM

Prove the image is what you think it is. Scan for known vulnerabilities, record exactly what's inside, and sign it so the runtime can refuse anything you didn't build.

**Pick:** hadolint + Trivy in CI; cosign once you have more than one service

| Tool | Status | Notes |
|---|---|---|
| hadolint | **PICK** | Lints Dockerfiles for common mistakes: unpinned versions, missing cleanup, root users. |
| Trivy | **PICK** | Scans images, filesystems and IaC for vulnerabilities and misconfigurations. Fails the build on critical findings. |
| Syft + Grype | alt | Syft generates an SBOM (the full parts list); Grype scans it for vulnerabilities. |
| Docker Scout | alt | Docker's built-in scanning and recommendations, integrated with Docker Hub and Desktop. |
| cosign / Sigstore | **PICK** | Sign images keylessly using your CI's identity. Clusters can then enforce signed-only deploys. |

---

## L4 · Registry

Where built images live, tagged and versioned, until something pulls them. Keep it close to where you run, and in the same cloud account if you can.

**Pick:** GHCR for GitHub-based projects; your cloud's registry in production

| Tool | Status | Notes |
|---|---|---|
| GitHub Container Registry | **PICK** | Free for public images, permissions tied to your repo, trivial to push from Actions. |
| ECR / Artifact Registry / ACR | **PICK** | The registry inside your cloud. Fast pulls, IAM permissions, built-in scanning. |
| Docker Hub | alt | The public default for open-source images. Anonymous pulls are rate-limited, which bites CI. |
| Harbor | alt | Self-hosted registry with scanning, signing and replication. CNCF graduated. |

---

## L3 · Local runtime

Running containers on your own machine for development and debugging, usually several at once: app, database, cache.

**Pick:** Docker Desktop or OrbStack, plus Docker Compose

| Tool | Status | Notes |
|---|---|---|
| Docker Desktop | **PICK** | The standard, on every OS. Free for personal and small-business use; paid for larger companies. |
| OrbStack | alt | Mac-only replacement for Docker Desktop. Much faster and lighter; paid for commercial use. |
| Podman Desktop | alt | Daemonless, rootless and fully open source. Docker-compatible CLI. |
| Colima / Rancher Desktop | alt | Free, open-source container runtimes for Mac and Linux. Colima is CLI-only and minimal. |
| Docker Compose | **PICK** | Define app, Postgres and Redis in one YAML file and start them together. The v2 plugin built into docker. |
| WSL 2 | alt | On Windows, Docker runs inside WSL's Linux VM. Keep project files in the WSL filesystem for speed. |

---

## L2 · Run target

Where the container actually runs in production. Pick the least platform that fits: every step toward Kubernetes buys control and costs operations time.

**Pick:** Serverless containers: Cloud Run, ECS Fargate or Azure Container Apps

### Serverless containers

| Tool | Status | Notes |
|---|---|---|
| Google Cloud Run | **PICK** | Deploy an image, get HTTPS and autoscaling, including down to zero. Generous free tier. |
| AWS ECS Fargate | **PICK** | Containers on AWS without managing servers. More setup than Cloud Run; the AWS default. |
| Azure Container Apps | alt | Azure's serverless containers, with scale to zero and Dapr built in. |

### PaaS & plain servers

| Tool | Status | Notes |
|---|---|---|
| Fly.io / Render / Railway | alt | Developer-friendly platforms: push an image or repo, get a URL. Great for side projects. |
| VPS + Compose + Caddy | alt | One Linux box running Docker Compose behind Caddy. Cheap, simple, and you own the patching. |
| Docker Swarm | ⚠️ edge | Built-in Docker clustering. Mirantis supports it through 2030, but it gets no major features. Fine to keep, not to start. |

---

## L1 · Orchestration

Kubernetes and its ecosystem, for when you run many services that need shared networking, policies and deployment patterns. Most single services don't need it.

**Pick:** A managed cluster (EKS, GKE, AKS), Helm, and Gateway API

| Tool | Status | Notes |
|---|---|---|
| Managed Kubernetes | **PICK** | EKS, GKE or AKS. Never run your own control plane unless that's your job. |
| Helm | **PICK** | Package and template Kubernetes manifests; installs most third-party software. |
| Kustomize | alt | Patch plain YAML per environment, no templating. Built into kubectl. |
| Gateway API | **PICK** | The successor to Ingress for routing traffic into a cluster. The recommended path since ingress-nginx retired. |
| k3s | alt | Lightweight Kubernetes in one binary. Edge devices, home labs, small clusters. |
| Nomad | alt | HashiCorp's simpler orchestrator for containers and plain binaries. Business Source License. |

---

## L0 · Infrastructure & delivery

The ground everything stands on, declared in code, and the pipeline that moves each commit through all the layers above automatically.

**Pick:** GitHub Actions for CI/CD; Terraform or OpenTofu for infrastructure

| Tool | Status | Notes |
|---|---|---|
| GitHub Actions | **PICK** | Build, scan, push and deploy on every merge. Authenticate to clouds with OIDC instead of stored keys. |
| Terraform / OpenTofu | **PICK** | Declare registries, services, networks and permissions in code. OpenTofu is the open-source fork. |
| Pulumi | alt | Infrastructure as real Python code instead of HCL. Natural for Python teams. |
| Argo CD / Flux | alt | GitOps for Kubernetes: the cluster continuously syncs itself to what's in a Git repo. |
| Renovate / Dependabot | **PICK** | Automated pull requests that bump base images, dependencies and actions. |

---

## Sidecars

Concerns that apply at every layer once the thing is running. They're where most production incidents actually come from.

### Config & secrets

| Tool | Status | Notes |
|---|---|---|
| Environment variables | **PICK** | Twelve-factor config: the same image runs everywhere, with behavior set by the environment. |
| pydantic-settings | **PICK** | Reads and validates those variables into typed settings at startup, so bad config fails fast. |
| Cloud secret managers | **PICK** | AWS Secrets Manager, Google Secret Manager, Azure Key Vault: inject at runtime, never bake. |
| SOPS / sealed secrets | alt | Keep encrypted secrets in Git and decrypt at deploy time. |

### Runtime health

| Tool | Status | Notes |
|---|---|---|
| Liveness & readiness | **PICK** | /healthz says the process is alive; /readyz says it can serve. Platforms restart or route on these. |
| Graceful shutdown | **PICK** | Handle SIGTERM, finish in-flight requests, then exit. Required for zero-downtime deploys. |
| OpenTelemetry | **PICK** | Traces and metrics leave the container through a collector; logs go to stdout. |

### Cost & scale

| Tool | Status | Notes |
|---|---|---|
| Scale to zero | **PICK** | Pay nothing when idle. The trade is a cold start on the first request. |
| Min instances | alt | Keep one warm instance for latency-sensitive services. |
| Right-size CPU & memory | **PICK** | Most Python services are over-provisioned. Measure, then shrink. |

---

## Rules of the road

- **Bake, don't pull.** Everything the service needs goes into the image at build time. An image is an immutable snapshot of one commit, so it starts fast and you can roll back to any version.
- **Dependencies before code.** Copy the lockfile and install first, then copy your source. Code changes then rebuild in seconds because the dependency layer stays cached.
- **Never bake secrets.** Anything copied into a layer stays in the image forever, even if a later step deletes it. Inject secrets at runtime.
- **One process per container.** Let the platform handle restarts and scaling instead of running a process manager inside.
- **Logs to stdout.** Write logs as JSON to standard output and let the platform collect them. No log files inside containers.
- **Pin by digest.** Tags like 3.15-slim move. Pin base images by digest and let Renovate or Dependabot bump them.

## The licensing footnotes

The container world keeps rewriting its terms. Docker Desktop is free for personal use and small businesses but paid above 250 employees or $10M revenue. HashiCorp moved Terraform and Nomad to the Business Source License in 2023, which spawned the OpenTofu fork. Broadcom ended Bitnami's free image catalog in September 2025. Before adopting any piece of this stack, read the current license, not the one you remember.

## Dog Solitude

The scrapyard. Dead, retired, removed or archived tools, by layer.

| Code | Short | Layer | Obsolete |
|---|---|---|---|
| L8 | base | Base image | CentOS images, Python 2 images, Debian stretch/buster-based Python tags |
| L6 | build | Build tooling | Docker legacy builder, kaniko (Google original, archived June 2025) |
| L4 | registry | Registry | Bitnami free image catalog (moved to bitnamilegacy, Sept 2025), Docker Hub free automated builds |
| L3 | local | Local runtime | docker-compose v1 (the Python one), Docker Machine, Docker Toolbox, boot2docker |
| L1 | orchestrate | Orchestration | ingress-nginx (retired March 2026), dockershim, PodSecurityPolicy, classic standalone Swarm, Mesos + Marathon |

## Sources

- [Ingress NGINX Retirement (Kubernetes blog)](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Kaniko lives on: Chainguard forks Google’s dumped tool (The New Stack)](https://thenewstack.io/kaniko-lives-on-chainguard-forks-googles-dumped-tool/)
- [Bitnami ends free, stable images (Linuxiac)](https://linuxiac.com/bitnami-ends-free-stable-images-users-forced-to-migrate-or-pay/)
- [Is Docker Swarm still safe in 2026? (Virtualization Howto)](https://www.virtualizationhowto.com/2026/03/is-docker-swarm-still-safe-in-2026/)
