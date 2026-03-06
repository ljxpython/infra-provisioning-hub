# Ollama + Docker Reproducible Setup Guide (Debian 12)

This document captures a reproducible setup for running Ollama in Docker with registry mirrors, plus optional NVIDIA GPU container runtime setup.

## 0) Scope and validated environment

- OS: Debian GNU/Linux 12 (bookworm)
- Docker: Installed and running
- Ollama image: `ollama/ollama:latest`
- Mirror strategy: Docker daemon registry mirrors

Note:
- GPU mode requires BOTH physical NVIDIA GPU and host NVIDIA driver (`nvidia-smi` must work on host).
- If no NVIDIA GPU is present, use CPU mode (works for RAG development and testing).

## 1) Configure Docker registry mirrors

Create or replace `/etc/docker/daemon.json` with:

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://hub-mirror.c.163.com"
  ]
}
```

Apply config:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
docker info | sed -n '/Registry Mirrors/,+6p'
```

Expected output should list the three mirror URLs.

## 2) Pull Ollama image

```bash
docker pull ollama/ollama:latest
```

If needed, fallback pull path:

```bash
docker pull m.daocloud.io/docker.io/ollama/ollama:latest
docker tag m.daocloud.io/docker.io/ollama/ollama:latest ollama/ollama:latest
```

## 3) Run Ollama container (CPU mode)

If an old broken container exists from a failed GPU start:

```bash
docker rm -f ollama || true
```

Start container:

```bash
docker run -d \
  --name ollama \
  -p 11434:11434 \
  -v ollama_data:/root/.ollama \
  --restart unless-stopped \
  ollama/ollama:latest
```

Verify service:

```bash
curl http://localhost:11434/api/tags
```

## 4) Optional: enable NVIDIA GPU for Docker

Run this only on machines that actually have an NVIDIA GPU.

### 4.1 Check hardware and host driver

```bash
lspci | grep -Ei 'vga|3d|display|nvidia'
nvidia-smi
```

Requirements:
- `lspci` shows NVIDIA device
- `nvidia-smi` returns GPU info (not command-not-found)

### 4.2 Install NVIDIA container toolkit

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /etc/apt/keyrings/nvidia-container-toolkit-keyring.gpg
sudo chmod a+r /etc/apt/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/etc/apt/keyrings/nvidia-container-toolkit-keyring.gpg] https://#' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list >/dev/null

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

### 4.3 Configure Docker runtime for NVIDIA

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
docker info | sed -n '/Runtimes/,+4p'
```

Expected output includes `nvidia` in runtimes.

### 4.4 Run Ollama with GPU

```bash
docker rm -f ollama || true

docker run -d \
  --name ollama \
  --gpus all \
  -p 11434:11434 \
  -v ollama_data:/root/.ollama \
  --restart unless-stopped \
  ollama/ollama:latest
```

GPU sanity test (optional):

```bash
docker run --rm --gpus all nvidia/cuda:12.3.1-base-ubuntu22.04 nvidia-smi
```

## 5) Pull model and test inference

```bash
docker exec -it ollama ollama pull qwen2.5:7b
docker exec -it ollama ollama run qwen2.5:7b
```

## 6) Common failures and fixes

### A) Pull timeout from Docker Hub

Symptom:
- `i/o timeout` when pulling `ollama/ollama:latest`

Fix:
- Ensure mirror config in `/etc/docker/daemon.json`
- Restart Docker
- Retry `docker pull ollama/ollama:latest`
- Use fallback path then retag

### B) `could not select device driver "" with capabilities: [[gpu]]`

Root cause:
- Docker NVIDIA runtime not installed/configured OR no NVIDIA GPU/driver on host

Fix:
- Install `nvidia-container-toolkit`
- Run `nvidia-ctk runtime configure --runtime=docker`
- Restart Docker
- Confirm `docker info` shows `nvidia`
- Confirm `nvidia-smi` works on host

### C) Container already exists

Symptom:
- `Conflict. The container name "/ollama" is already in use`

Fix:

```bash
docker rm -f ollama
```

## 7) RAG app endpoint

Use this base URL in your RAG service code:

- Local: `http://localhost:11434`
- Remote: `http://<server-ip>:11434`

