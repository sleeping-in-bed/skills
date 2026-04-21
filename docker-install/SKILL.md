---
name: docker-install
description: Install, set up, configure, verify, troubleshoot, update, reinstall, or remove Docker Engine on Linux or WSL Ubuntu using Docker's official docker/docker-install convenience-script repository. Use for any natural-language request about installing Docker, such as "install docker", "setup docker", "set up Docker Engine", "装 Docker", "安装 docker", "在 WSL 里装 docker", "给 Ubuntu 安装 Docker", "用 docker/docker-install 装 Docker", "get.docker.com", "Docker CE on WSL", or quick Docker Engine setup. If an NVIDIA GPU is detected in Linux or WSL, also install and configure NVIDIA Container Toolkit for Docker GPU containers unless the user opts out. Avoid for Docker Desktop-only setup unless comparing alternatives or the user explicitly wants Desktop instead of Engine inside WSL/Linux.
---

# Docker Install

## Purpose

Use Docker's official `docker/docker-install` repository to install the latest stable Docker Engine packages on supported Linux distributions. The upstream repository states this script is a convenience for quickly installing Docker CE and is not recommended as a production deployment dependency.

Prefer this skill for local development, WSL Ubuntu, test machines, and quick setup. For production or pinned versions, use Docker's distro-specific package-install documentation instead.

## Before Installing

1. Identify where Docker should run:
   - WSL Ubuntu/Linux distro: use the workflow below.
   - Windows host with Docker Desktop: install/configure Docker Desktop instead; do not also install Docker Engine inside WSL unless the user explicitly wants an independent Linux daemon.
2. Check existing Docker:
   ```bash
   command -v docker && docker --version
   docker context ls 2>/dev/null || true
   ```
3. On WSL, check the distro and service manager:
   ```bash
   cat /etc/os-release
   ps -p 1 -o comm=
   ```
4. Do not pipe remote scripts straight into `sh`. Clone the official repository, run `--dry-run`, then install.

## Install From The Repository

Run inside the target Linux or WSL distro:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl git

workdir="$(mktemp -d)"
git clone --depth=1 https://github.com/docker/docker-install.git "$workdir/docker-install"
cd "$workdir/docker-install"

sudo sh install.sh --dry-run
sudo sh install.sh
```

The same upstream script is available through `https://get.docker.com`, but prefer the repository clone when the user asked to use `docker/docker-install`.

## Start Docker

Use systemd when available:

```bash
sudo systemctl enable --now docker
```

If systemd is not PID 1, common in some WSL setups, use:

```bash
sudo service docker start
```

For WSL users who want Docker to start more naturally with the distro, enable systemd:

```bash
printf '[boot]\nsystemd=true\n' | sudo tee /etc/wsl.conf
```

Then from Windows PowerShell:

```powershell
wsl --shutdown
```

Reopen WSL and run:

```bash
sudo systemctl enable --now docker
```

## Non-Root Docker Use

Add the current Linux user to the `docker` group only after explaining that members of this group effectively have root-equivalent control over Docker:

```bash
sudo usermod -aG docker "$USER"
```

Apply group membership by closing/reopening the WSL shell, or temporarily:

```bash
newgrp docker
```

## Verify

```bash
docker --version
docker compose version
docker info
docker run --rm hello-world
```

On WSL, also check Windows-side visibility only if the user expects Windows tools to use the same Docker daemon. Docker Engine installed inside WSL is separate from Docker Desktop.

## GPU Containers

The `docker/docker-install` script installs Docker Engine; it does not configure NVIDIA GPU container runtime by itself. If an NVIDIA GPU is detected, or the user needs `docker run --gpus all`, install NVIDIA Container Toolkit using NVIDIA's official install guide:

- NVIDIA Container Toolkit install guide: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
- CUDA on WSL guide: https://docs.nvidia.com/cuda/wsl-user-guide/index.html

First verify Linux or WSL can see the GPU:

```bash
nvidia-smi
ls -l /dev/dxg
```

On WSL, do not install a Linux NVIDIA display driver. The CUDA on WSL guide says the Windows NVIDIA driver is the driver WSL uses, and Linux driver packages inside WSL can break the mapped driver setup.

For Ubuntu/Debian with apt, use the official NVIDIA package repository:

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends ca-certificates curl gnupg2

curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor --yes -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
apt-cache madison nvidia-container-toolkit | sed -n '1,10p'
export NVIDIA_CONTAINER_TOOLKIT_VERSION="${NVIDIA_CONTAINER_TOOLKIT_VERSION:-1.19.0-1}"
sudo apt-get install -y \
  nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
  nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
  libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
  libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

Configure Docker and restart it:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

If systemd is not available, restart Docker with the distro's service command:

```bash
sudo service docker restart
```

Verify that Docker sees the NVIDIA runtime and can pass the GPU through:

```bash
docker info | sed -n '/Runtimes:/,/Default Runtime:/p'
docker run --rm --gpus all nvidia/cuda:12.6.3-base-ubuntu24.04 nvidia-smi
```

Expected indicators:

- `docker info` includes `nvidia` in `Runtimes`.
- The CUDA container prints `nvidia-smi` output showing the host GPU.
- `/etc/docker/daemon.json` contains a `runtimes.nvidia.path` value of `nvidia-container-runtime`.

Installation notes from real WSL setup:

- NVIDIA's official `nvidia-container-toolkit` package installed version `1.19.0-1` with `libnvidia-container1`, `libnvidia-container-tools`, and `nvidia-container-toolkit-base`.
- `nvidia-ctk runtime configure --runtime=docker` creates or updates `/etc/docker/daemon.json`, then Docker must be restarted.
- Docker Hub pulls can intermittently fail with `EOF` or `TLS handshake timeout`; retry before changing configuration.
- Avoid `set -o pipefail` with `apt-cache madison nvidia-container-toolkit | head -10`; `head` can close the pipe early and abort the script. Use `sed -n '1,10p'` instead.

## Troubleshooting

- `Cannot connect to the Docker daemon`: start Docker with `sudo systemctl start docker` or `sudo service docker start`.
- Permission denied on `/var/run/docker.sock`: use `sudo docker ...`, or add the user to the `docker` group and reopen the shell.
- WSL website/container disappears after reboot: set container restart policy and ensure Docker daemon starts again:
  ```bash
  docker update --restart unless-stopped <container>
  ```
- Existing Docker Desktop context confusion: inspect `docker context ls`; choose whether the user wants Docker Desktop's daemon or the independent WSL Docker Engine.
- Reinstall/cleanup requests: avoid destructive package removal unless the user explicitly asks; inspect existing containers/images/volumes first.

## Uninstall

Only uninstall when the user asks for removal. Preserve data unless they explicitly ask to delete it.

```bash
sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
sudo apt-get autoremove -y
```

Docker data usually remains under `/var/lib/docker`. Do not delete it without explicit confirmation:

```bash
sudo rm -rf /var/lib/docker /var/lib/containerd
```
