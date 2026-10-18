# HOWTO: Install Docker for use with WSL and VS Code

How to install Docker Desktop on Windows and use it exclusively from WSL 2, plus some background on registries, tags, labels and packaging. VS Code setup is still a to-do (see the last section).

WSL itself is covered in [HOWTO_wsl_setup.md](HOWTO_wsl_setup.md). This HOWTO assumes it's done.

## How it fits together

Docker Desktop is built to work with WSL 2, and on systems that support WSL 2 it uses the WSL 2 engine by default. Docker runs inside its own `docker-desktop` WSL distro, kept separate from your other distros. It only talks to a distro (like Ubuntu) once you turn on WSL integration for that distro, which is what makes the `docker` command work from your Ubuntu terminal.

## Install

### 1. Install WSL and a distro

If not already done, in PowerShell **as administrator**:

```
wsl --install
```

This installs WSL 2 with Ubuntu. Restart, then open Ubuntu once to create your Linux username and password. Full details in [HOWTO_wsl_setup.md](HOWTO_wsl_setup.md).

### 2. Update WSL and confirm version 2

Docker requires WSL 2.1.5 or later and recommends the latest. In PowerShell:

```
wsl --update
wsl -l -v
```

Your distro should show `VERSION 2`. If it shows 1:

```
wsl --set-version Ubuntu 2
```

To make 2 the default for future distros:

```
wsl --set-default-version 2
```

### 3. Remove any Docker installed inside Linux

Docker Desktop conflicts with a Docker Engine or Docker CLI installed directly inside a WSL distro. In Ubuntu (harmless if nothing is installed):

```
sudo apt remove docker docker-engine docker.io containerd runc
```

### 4. Install Docker Desktop for Windows

Download Docker Desktop for Windows from [docker.com](https://www.docker.com/products/docker-desktop/) and run the installer. If it asks about WSL 2, leave that option on.

### 5. Start Docker Desktop and check the engine

Start Docker Desktop from the Windows Start menu. Under **Settings > General**, **Use WSL 2 based engine** should be on. On systems that support WSL 2 it's on by default and may not even be shown.

### 6. Turn on WSL integration for your distro

Go to **Settings > Resources > WSL Integration**, turn on your distro (e.g. Ubuntu), and select **Apply**.

If **WSL Integration** isn't listed under **Resources**, Docker is probably in Windows container mode. Open the Docker menu in the taskbar and select **Switch to Linux containers**.

### 7. Test it from Ubuntu

Open a new Ubuntu terminal and run:

```
docker run hello-world
```

No `sudo` and no `docker` group setup needed. Docker Desktop handles that.

## Habits to adopt

- **Keep projects in your Linux home, not under `/mnt/c/`.** File access across the Windows/Linux boundary is much slower, which makes bind mounts and builds crawl.
- **Docker Desktop must be running in Windows** for `docker` to work in WSL.
- **Consider WSL's `autoMemoryReclaim` setting.** It lets Windows take back RAM that WSL holds onto after image builds finish.
- By default, Docker Desktop stores its WSL 2 engine data at `C:\Users\[USERNAME]\AppData\Local\Docker\wsl`. To move it, go to **Settings > Resources > Advanced**.

### What to call the projects folder

There's no official convention. `~/projects` is one common choice; others are `~/code`, `~/src`, `~/dev`, `~/repos` and `~/workspace`. The name doesn't matter. What matters is that it lives under your Linux home (`~`). Keep it short since you'll type it a lot.

## Where to put your own images (registries)

| Registry | Good for | Free tier notes |
|---|---|---|
| **Docker Hub** | Public images; the default everyone knows | Unlimited public repos, but only **1 private repo** (up to 2 GB) on the free Personal plan. A repo holds one image and its tags, so that's one private image. Pro removes the limit. |
| **GitHub Container Registry** (`ghcr.io`) | Private images for free; code already on GitHub | GitHub currently lists container storage and bandwidth as free, with advance notice promised before that changes. Sits next to the source and works directly with GitHub Actions. |
| **Cloud registries** (AWS ECR, Google Artifact Registry, Azure ACR) | Deploying to that cloud | Overkill for personal use, except for learning (below). |

Short version: public images → Docker Hub. Private images, or code on GitHub → `ghcr.io`.

### Using AWS ECR to learn ECR

Worth it if the goal is learning AWS. ECR itself is fairly thin (push, pull, permissions). Most of what you learn comes from what's around it: IAM roles and policies, logging Docker in with the AWS CLI, and pulling images into ECS, Lambda or EKS.

Cost for a learning setup is small but not zero:

- **Storage:** private repos are about $0.10 per GB-month, with 500 MB/month free for the first 12 months.
- **New account credits:** accounts created since July 15, 2025 get up to $200 in Free Tier credits, usable within 12 months.
- **Pulls to your own machine:** traffic within one AWS region is free, but pulling images down to your WSL machine is internet data transfer and is billed. Small images keep this to pennies.

Before starting:

- Set up an AWS Budget alert (even $5).
- Add a lifecycle policy to the repo that keeps only the last few images.
- Delete the repo and anything else you created when you're done.

Plan: keep everyday images on Docker Hub or `ghcr.io`, and use ECR for exercises like pushing an image and running it on ECS Fargate or Lambda.

## Tags vs labels

A **tag** is a name for finding an image. A **label** is a note stored inside an object.

**Tag**

- Part of the image's address: in `myapp:1.2.0`, `1.2.0` is the tag. No tag means `latest`.
- Lives outside the image, in your local image list or the registry. One image can have several tags (`1.2.0`, `1.2`, `latest`).
- Movable. Push a new image as `latest` and `latest` now points to the new image.
- Used to pull, push and run: `docker run myapp:1.2.0`.

**Label**

- Key-value metadata attached to a Docker object: images, containers, local daemons, volumes and networks.
- Set with `LABEL` in a Dockerfile, or `--label` when creating a container, volume or network.
- Used for things like organizing images, recording license info, and noting how containers, volumes and networks relate.
- Fixed for the life of the object. Changing one means recreating the object.
- Read with `docker inspect`, filter with e.g. `docker image ls --filter "label=com.example.version"`.

Common pattern: tag the image `myapp:1.2.0` so people can pull it, and add the label `org.opencontainers.image.version=1.2.0` so the version stays recorded inside the image even if someone retags it.

## Libraries don't go in containers

A container packages something that runs. A library (a Python package, a Java JAR) gets pulled into someone else's project at build time, so it's published to a package registry in its native format:

- **Python:** a wheel on PyPI (or a private index), installed with `pip`
- **Java:** a JAR on Maven Central (or a private Maven repo), pulled in by Maven or Gradle
- **JavaScript:** a package on npm

The container comes later, in the app that uses the library. That app's Dockerfile runs `pip install yourlib` or a Gradle build, and the library ends up inside its image as a dependency.

Where containers do show up near library work:

- **Base images** that bundle a runtime plus common libraries, so apps start from a known setup.
- **Build/test environments** for testing a library across Python or Java versions. What you publish is still the wheel or JAR.
- **Command-line tools** built on a library, which can also ship as an image since they run.

If you're on GitHub anyway, GitHub Packages can host Maven, npm and container images in one place.

Side note: the name "wheel" is permanent (it's in the spec, PEP 427, the `.whl` extension and `pip`). It's a Monty Python joke: PyPI was nicknamed "the Cheese Shop," and cheese comes in wheels. The format it replaced was the "egg."

## VS Code

**TODO.** Not covered yet. To fill in:

- [ ] Install the WSL extension and open projects from WSL (`code .` from Ubuntu)
- [ ] Install the Docker / Container Tools extension
- [ ] Dev Containers: when and how to use them
- [ ] Confirm VS Code sees Docker Desktop's engine from inside WSL

See also [HOWTO_claude_code_in_wsl_and_vs_code.md](HOWTO_claude_code_in_wsl_and_vs_code.md) for VS Code + WSL basics.

## Sources

- [Docker Desktop WSL 2 backend on Windows](https://docs.docker.com/desktop/features/wsl/)
- [Docker Hub storage usage and limits](https://docs.docker.com/docker-hub/usage/storage)
- [Docker Hub usage and limits](https://docs.docker.com/docker-hub/usage/)
- [Docker object labels](https://docs.docker.com/engine/manage-resources/labels/)
- [Top 8 Container Registries for 2026](https://guptadeepak.com/tools/top-8-container-registries-2026/)
- [Amazon ECR Pricing: What $0.10/GB Doesn't Tell You](https://cloudburn.io/blog/amazon-ecr-pricing)
