# Guide: Setting Up a Custom ComfyUI Docker Environment for RTX 3060

This guide walks you through the process of creating a specialized Docker setup for ComfyUI, optimized for a NVIDIA RTX 3060 GPU. It automates forking a repository, customizing the Docker configuration, adding a comprehensive set of custom nodes, and setting up a GitHub Actions workflow to build and publish your custom image.

## Prerequisites

* You have the `gh` (GitHub CLI) installed and authenticated (`gh auth login`).
* You have Docker installed.
* You have `git` installed.

## Instructions

Follow these steps to set up your environment.

### Step 1: Set Your GitHub Username

You need to replace the placeholder `your-github-username` in the script below with your actual GitHub username.

### Step 2: Run the Setup Script

This script performs all the necessary actions. Save it as `setup.sh`, make it executable with `chmod +x setup.sh`, and run it with `./setup.sh`.

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- Variables –-
# !!! IMPORTANT !!!
# !!! Replace 'your-github-username' with your actual GitHub username. !!!
GITHUB_USER="your-github-username"

# --- Configuration ---
FORK_ORG="${GITHUB_USER}"
ORIG_REPO="YanWenKun/ComfyUI-Docker"
FORK_REPO="${FORK_ORG}/ComfyUI-Docker"
CLONE_DIR="ComfyUI-Docker-RTX3060"
CUDA_TAG="cu126-slim"

# --- Execution ---

echo "Step 1: Forking repository..."
# Forks the original repository to your GitHub account.
gh repo fork "${ORIG_REPO}" --org "${FORK_ORG}" --clone=false

echo "Step 2: Cloning your fork locally..."
# Clones your newly created fork to a local directory.
gh repo clone "${FORK_REPO}" "${CLONE_DIR}"
cd "${CLONE_DIR}"

echo "Step 3: Adding upstream remote and creating a new branch..."
# Adds the original repository as an 'upstream' remote to pull future updates.
git remote add upstream "https://github.com/${ORIG_REPO}.git"
git fetch upstream
# Creates a new branch 'rtx3060-config' to store your custom configuration.
git checkout -b rtx3060-config upstream/main

echo "Step 4: Modifying Dockerfile for RTX3060 optimizations..."
# Modifies the Dockerfile to use a specific CUDA version and adds runtime arguments
# optimized for a GPU with medium VRAM, like the RTX 3060.
sed -i \
  -e "s|FROM .*:.*|FROM ghcr.io/yanwk/comfyui-boot:${CUDA_TAG}|g" \
  -e '/ENTRYPOINT \[/c\ENTRYPOINT [ "python3", "-u", "start.py", "--enable-sageattention", "--enable-triton", "--medvram", "--no-half-vae" ]' \
  Dockerfile

echo "Step 5: Writing docker-compose.yml..."
# Creates a docker-compose.yml file for easy container management.
cat > docker-compose.yml << EOF
version: "3.8"
services:
  comfyui:
    image: ghcr.io/${FORK_REPO}:${CUDA_TAG}
    build: .
    container_name: comfyui_rtx3060
    ports:
      - "8188:8188"
    volumes:
      - ./data:/root
      - ./custom-nodes:/opt/comfyui/custom_nodes
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
EOF

echo "Step 6: Cloning all required custom node repositories..."
# Creates the directory for custom nodes and clones a curated list of popular
# and useful nodes for image/video generation, model management, and more.
mkdir -p custom-nodes
cd custom-nodes

# Quantization and Video
git clone https://github.com/city96/ComfyUI-GGUF.git
git clone https://github.com/mit-han-lab/nunchaku.git
git clone https://github.com/kijai/ComfyUI-WanVideoWrapper.git
git clone https://github.com/liusida/top-100-comfyui.git

# Additional nodes
git clone https://github.com/dpbit/ComfyUI-FluxNodes.git
git clone https://github.com/krea-ai/ComfyUI-Krea.git
git clone https://github.com/comfyanonymous/ComfyUI-PythonScripts.git

# Video generation nodes
git clone https://github.com/Lightricks/ComfyUI-LTXVideo.git
git clone https://github.com/liusida/ComfyUI-HunyuanVideo.git

# ControlNet nodes and management
git clone https://github.com/Fannovel16/comfyui_controlnet_aux.git
git clone https://github.com/Kosinkadink/ComfyUI-Advanced-ControlNet.git

# LoRA model management
git clone https://github.com/willmiao/ComfyUI-Lora-Manager.git

cd ..

echo "Step 7: Configuring ComfyUI-Manager for convenience..."
# Clones the ComfyUI-Manager if it's not already present and sets its
# security configuration to 'weak'. This allows for easier management of custom
# nodes from within the ComfyUI interface.
MANAGER_DIR="./custom-nodes/ComfyUI-Manager"
if [ ! -d "${MANAGER_DIR}" ]; then
  echo "Cloning ComfyUI-Manager..."
  git clone https://github.com/comfyanonymous/ComfyUI-Manager.git "${MANAGER_DIR}"
fi

mkdir -p "${MANAGER_DIR}/config.ini"
echo -e "[security]\nsecurity=weak" > "${MANAGER_DIR}/config.ini"

echo "Step 8: Committing and pushing changes..."
# Commits all your local changes (Dockerfile, docker-compose, custom nodes)
# and pushes them to the 'rtx3060-config' branch on your fork.
git add Dockerfile docker-compose.yml custom-nodes
git commit -m "Enable RTX3060 optimizations, pin CUDA ${CUDA_TAG}, add full node set + weak security config"
git push --set-upstream origin rtx3060-config

echo "Step 9: Creating GitHub Actions workflow for Docker build and publish..."
# Sets up a GitHub Actions workflow that will automatically build your custom
# Dockerfile and publish the resulting image to the GitHub Container Registry (ghcr.io).
mkdir -p .github/workflows
cat > .github/workflows/docker-publish.yml << EOF
name: Build & Publish Docker image

on:
  push:
    branches: [ rtx3060-config, main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/setup-qemu-action@v2
      - uses: docker/setup-buildx-action@v2
      - name: Login to GHCR
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ghcr.io/${FORK_REPO}:${CUDA_TAG},ghcr.io/${FORK_REPO}:latest
EOF

git add .github/workflows/docker-publish.yml
git commit -m "Add GitHub Actions workflow for Docker build and publish"
git push

echo "Setup complete. Create a PR from rtx3060-config branch to main to finalize."
```

## What Happens Next?

1.  **Automatic Docker Build**: Pushing the `docker-publish.yml` workflow file to your repository will trigger a new GitHub Action. You can view its progress under the "Actions" tab of your forked repository.
2.  **Published Image**: Once the action completes successfully, your custom Docker image will be available at `ghcr.io/your-github-username/ComfyUI-Docker:cu126-slim`.
3.  **Pull Request**: As the final message suggests, you can create a pull request from your `rtx3060-config` branch to your `main` branch. This merges your changes, and the workflow will run again, publishing the image with the `:latest` tag as well.

## How to Use Your Custom Image

Once the image is published, you can run it using Docker Compose:

```bash
cd ComfyUI-Docker-RTX3060
docker-compose up -d
```

You can then access the ComfyUI interface by navigating to `http://localhost:8188` in your web browser.
