# 🚀 Usage Guide — Low-Resource CI/CD Blueprint

**Author:** Wildan Fathir Qinthara — Universitas Sebelas April (UNSAP)
**Stack:** GitHub Actions · Docker · GitHub Container Registry (GHCR)
**Target Environment:** Low-resource VPS (1 vCPU, 1 GB RAM)

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Part I — Host Server Preparation](#part-i--host-server-preparation)
   - [1.1 SWAP Memory Allocation (OOM Prevention)](#11-swap-memory-allocation-oom-prevention)
   - [1.2 Docker Engine Installation](#12-docker-engine-installation)
   - [1.3 Non-Root SSH User Configuration](#13-non-root-ssh-user-configuration)
3. [Part II — Project Dockerfile Requirements](#part-ii--project-dockerfile-requirements)
4. [Part III — Triggering a Deployment](#part-iii--triggering-a-deployment)
5. [Part IV — Pipeline Internals: Build → Push → Deploy](#part-iv--pipeline-internals-build--push--deploy)
6. [Part V — Discord ChatOps Monitoring](#part-v--discord-chatops-monitoring)
7. [Troubleshooting Reference](#troubleshooting-reference)

---

## Architecture Overview

This blueprint implements a **tag-triggered, cloud-offloaded CI/CD pipeline** designed specifically for low-resource server environments where running a build on the server itself would exhaust available RAM and CPU.

```
Developer Machine
      │
      │  git push origin v1.0.0-prod
      ▼
GitHub Repository ──── Triggers ────► GitHub Actions Runner (Cloud)
                                              │
                              ┌───────────────┼───────────────────┐
                              │               │                   │
                         Checkout      Install Deps         Build Docker
                         Source         & Compile            Image (via
                          Code           Assets              Buildx/Kit)
                              │               │                   │
                              └───────────────┴─────────┬─────────┘
                                                        │
                                                  Push Image to
                                                  GHCR Registry
                                                        │
                                                  SSH into VPS
                                                        │
                                          ┌─────────────▼──────────────┐
                                          │         Target VPS         │
                                          │  (1 vCPU · 1 GB RAM)      │
                                          │                            │
                                          │  docker pull <image>       │
                                          │  docker stop/rm old        │
                                          │  docker run new            │
                                          │  permission fix            │
                                          │  image prune               │
                                          └────────────────────────────┘
                                                        │
                                               Discord Notification
                                              (Success / Failure)
```

**Key design principle:** The computationally expensive build step runs entirely on GitHub's infrastructure, while the VPS only executes lightweight `docker pull` and `docker run` commands — keeping server resource consumption minimal.

---

## Part I — Host Server Preparation

All commands in this section must be run on the **target VPS** via an SSH terminal session. Connect to your server before proceeding:

```bash
ssh YOUR_USER@YOUR_SERVER_IP
```

---

### 1.1 SWAP Memory Allocation (OOM Prevention)

#### Why SWAP is Critical on Low-Resource Servers

A VPS with only **1 GB of physical RAM** is highly susceptible to **Out-Of-Memory (OOM) kill events** during the container initialization phase. When Docker pulls and starts a new container image, the Linux kernel may abruptly terminate processes — including the Docker daemon itself — if physical RAM is exhausted. This manifests as a failed deployment with no useful error message.

**SWAP space** is a designated area on the server's disk that the Linux kernel uses as a virtual RAM overflow buffer. While disk I/O is significantly slower than RAM, it prevents OOM crashes during memory pressure spikes.

**Recommended:** Allocate **2 GB** of SWAP (twice the physical RAM) for a 1 GB RAM server.

#### Step-by-Step SWAP Allocation Procedure

**Step 1: Verify current SWAP status.**

```bash
free -h
```

Expected output if no SWAP is configured:

```
               total        used        free      shared  buff/cache   available
Mem:           985Mi       312Mi       401Mi       1.2Mi       271Mi       520Mi
Swap:            0B          0B          0B
```

**Step 2: Allocate a 2 GB SWAP file using `fallocate`.**

```bash
sudo fallocate -l 2G /swapfile
```

> **Note:** `fallocate` instantly pre-allocates the file space on supported filesystems (ext4, XFS). If your filesystem does not support `fallocate`, use `dd` instead:
>
> ```bash
> sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress
> ```

**Step 3: Restrict file permissions to root-only access.**

This is a mandatory security step. A SWAP file must be readable only by root to prevent other users from reading potentially sensitive memory contents.

```bash
sudo chmod 600 /swapfile
```

Verify the permissions are correct:

```bash
ls -lh /swapfile
# Expected output: -rw------- 1 root root 2.0G ...
```

**Step 4: Initialize the file as a Linux SWAP area.**

```bash
sudo mkswap /swapfile
```

Expected output:

```
Setting up swapspace version 1, size = 2 GiB (2147479552 bytes)
no label, UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

**Step 5: Activate the SWAP space immediately (current session).**

```bash
sudo swapon /swapfile
```

**Step 6: Verify SWAP is now active.**

```bash
free -h
```

Expected output after activation:

```
               total        used        free      shared  buff/cache   available
Mem:           985Mi       318Mi       395Mi       1.2Mi       271Mi       514Mi
Swap:          2.0Gi          0B       2.0Gi
```

**Step 7: Persist SWAP across server reboots via `/etc/fstab`.**

Without this step, the SWAP space will be deactivated after every server restart.

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify the entry was appended correctly:

```bash
cat /etc/fstab | grep swapfile
# Expected output: /swapfile none swap sw 0 0
```

**Step 8: (Optional) Tune SWAP aggressiveness with `vm.swappiness`.**

The `swappiness` kernel parameter controls how aggressively the kernel moves memory pages to SWAP (0 = avoid SWAP; 100 = aggressive). For a server where SWAP is a safety net (not primary memory), a value of `10` is recommended:

```bash
# Apply immediately (non-persistent):
sudo sysctl vm.swappiness=10

# Persist across reboots:
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```

---

### 1.2 Docker Engine Installation

This pipeline requires the **official Docker Engine** runtime — not the `docker.io` package from Ubuntu's default repository, which is often significantly outdated.

The official method uses Docker's convenience installation script, which automatically detects your OS distribution and installs the correct, up-to-date packages.

#### Installation via Convenience Script

**Step 1: Download and execute the official Docker installation script.**

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

The script will:

1. Detect your Linux distribution (Ubuntu, Debian, CentOS, etc.)
2. Add Docker's official GPG key and package repository
3. Install `docker-ce`, `docker-ce-cli`, `containerd.io`, and `docker-compose-plugin`

**Step 2: Start and enable the Docker daemon.**

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

This ensures Docker starts automatically on every server reboot.

**Step 3: Verify the installation.**

```bash
docker --version
# Expected output: Docker version 26.x.x, build xxxxxxx

sudo docker run hello-world
# Expected: A "Hello from Docker!" success message
```

> **Note:** The convenience script is suitable for development and single-node production deployments. For enterprise environments, refer to the official Docker documentation for repository-based installation: https://docs.docker.com/engine/install/

---

### 1.3 Non-Root SSH User Configuration

The GitHub Actions SSH deployment step requires a user account on the VPS that can:

1. **Authenticate via SSH key** (passwordless login)
2. **Execute Docker commands** without `sudo`

Running Docker as `root` is a security anti-pattern in production. This section configures a dedicated, least-privilege deployment user.

#### Option A: Configure the Existing Default User (e.g., `ubuntu`)

If your VPS already has a default non-root user (e.g., `ubuntu` on Ubuntu distributions):

**Step 1: Add the user to the `docker` group.**

```bash
sudo usermod -aG docker ubuntu
# Replace 'ubuntu' with your actual username (e.g., 'deploy', 'admin')
```

**Step 2: Apply the group change without logging out.**

```bash
newgrp docker
```

**Step 3: Verify the user can run Docker without sudo.**

```bash
docker ps
# Expected: A table of running containers (or an empty list) — no "permission denied" error
```

**Step 4: Verify SSH key authentication is configured.**

The public key for the SSH key pair used in `SERVER_SSH_KEY` must be present in the user's `authorized_keys` file:

```bash
cat ~/.ssh/authorized_keys
# The public key content from your id_ed25519_deploy.pub should be listed here
```

If missing, add it manually:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "YOUR_PUBLIC_KEY_CONTENT_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

#### Option B: Create a Dedicated Deployment User (Recommended for Production)

For strict security isolation, create a purpose-specific user account:

**Step 1: Create the new deployment user.**

```bash
sudo useradd -m -s /bin/bash deploy
```

**Step 2: Add the user to the `docker` group.**

```bash
sudo usermod -aG docker deploy
```

**Step 3: Configure SSH key authentication for the new user.**

```bash
sudo mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh

# Paste the PUBLIC key content from your id_ed25519_deploy.pub file:
echo "YOUR_PUBLIC_KEY_CONTENT_HERE" | sudo tee /home/deploy/.ssh/authorized_keys

sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh
```

**Step 4: Test the SSH connection from your local machine.**

```bash
ssh -i ~/.ssh/id_ed25519_deploy deploy@YOUR_SERVER_IP
```

A successful connection confirms the `SERVER_USER` and `SERVER_SSH_KEY` secrets are correctly configured.

---

## Part II — Project Dockerfile Requirements

The pipeline builds a Docker image from a `Dockerfile` located in the **root directory** of your repository. Below are the structural requirements and best-practice recommendations.

#### Minimum Viable Dockerfile Structure

Every `Dockerfile` in this pipeline must follow multi-stage build conventions where possible to minimize the final production image size (critical for low-bandwidth server pull operations).

**Example: Laravel / PHP Application**

```dockerfile
# ============================================================
# Stage 1: Asset Builder
# Compiles frontend assets using Node.js. This stage is
# discarded in the final image, keeping the production
# image free of Node.js and its large node_modules directory.
# ============================================================
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY . .
RUN npm run build

# ============================================================
# Stage 2: Production Runtime
# The final image contains only what's needed to serve the
# application, resulting in a smaller, more secure image.
# ============================================================
FROM php:8.2-fpm-alpine AS production

WORKDIR /var/www/html

# Install required PHP extensions
RUN docker-php-ext-install pdo pdo_mysql opcache

# Copy application files
COPY --chown=www-data:www-data . .

# Copy compiled frontend assets from the builder stage
COPY --from=builder --chown=www-data:www-data /app/public/build ./public/build

# Install PHP production dependencies via Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
RUN composer install --no-dev --optimize-autoloader --no-interaction

# Set correct file permissions for Laravel's writable directories
RUN chmod -R 775 storage bootstrap/cache \
    && chown -R www-data:www-data storage bootstrap/cache

EXPOSE 9000
CMD ["php-fpm"]
```

**Example: Python / FastAPI Application**

```dockerfile
FROM python:3.11-slim AS production

WORKDIR /app

# Copy only the requirements file first for Docker layer caching
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Example: Go Application**

```dockerfile
# Stage 1: Compiler
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Stage 2: Minimal runtime image
FROM alpine:latest AS production

RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .

EXPOSE 8080
CMD ["./main"]
```

#### Dockerfile Placement Requirements

| Item                | Requirement                                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------------------- |
| **Location**        | Must be at the repository root (same level as `README.md`, `composer.json`, etc.)                       |
| **Filename**        | Must be named exactly `Dockerfile` (capital D, no extension)                                            |
| **`.dockerignore`** | Strongly recommended to exclude `node_modules/`, `.git/`, `tests/`, `*.md` to reduce build context size |

**Recommended `.dockerignore` file:**

```
.git
.github
node_modules
vendor
tests
*.test
*.md
.env.example
.env.local
Dockerfile
docker-compose*.yml
```

---

## Part III — Triggering a Deployment

This pipeline uses an **immutable, tag-based release strategy**. Deployments are triggered exclusively by pushing a Git tag matching the pattern `v*-prod` to the remote repository. This convention provides several production safety benefits:

- **Immutability:** Every deployment is tied to a specific, permanent commit snapshot.
- **Auditability:** The deployment history (via Git tags) serves as a transparent release changelog.
- **No accidental deploys:** Branch pushes (including `main`/`master`) do **not** trigger deployments, preventing unintended production releases from merge commits.

#### Full Deployment Workflow (Step by Step)

**Step 1: Ensure your changes are committed and pushed to the main branch.**

```bash
# Stage all changes
git add .

# Commit with a descriptive message
git commit -m "feat: add user authentication module"

# Push to the remote main branch
git push origin main
```

**Step 2: Create a versioned production Git tag.**

Tag names must follow **Semantic Versioning** (`MAJOR.MINOR.PATCH`) with the `-prod` suffix:

```bash
git tag v1.0.0-prod
```

**Tag naming conventions:**

| Scenario                        | Tag Example   |
| ------------------------------- | ------------- |
| First production release        | `v1.0.0-prod` |
| Bug fix release                 | `v1.0.1-prod` |
| Minor feature release           | `v1.1.0-prod` |
| Major version / breaking change | `v2.0.0-prod` |

**Step 3: Push the tag to the remote repository to trigger the pipeline.**

```bash
git push origin v1.0.0-prod
```

> **Important:** Pushing the tag (not the commit) is what triggers the GitHub Actions workflow. The `git push origin main` command alone does **not** start the pipeline.

**Step 4: Monitor the workflow execution.**

Navigate to your repository on GitHub → click the **Actions** tab. You will see a new workflow run titled `Build & Push & Deploy Application` with the tag name as its identifier.

#### Complete One-Liner Deployment Command

Once familiar with the workflow, you can chain the steps for efficiency:

```bash
# Commit, tag, and deploy in sequence
git add . && \
git commit -m "release: v1.0.0-prod" && \
git push origin main && \
git tag v1.0.0-prod && \
git push origin v1.0.0-prod
```

#### Deleting and Re-Pushing a Tag (Hotfix Redeploy)

If you need to redeploy the same semantic version after a hotfix without incrementing the version number, you can delete and recreate the tag:

```bash
# Delete the tag locally
git tag -d v1.0.0-prod

# Delete the tag from the remote repository
git push origin --delete v1.0.0-prod

# Apply your hotfix commit, then recreate and push the tag
git add .
git commit -m "hotfix: critical authentication bypass fix"
git tag v1.0.0-prod
git push origin v1.0.0-prod
```

> **Note:** This practice is not recommended for truly public/immutable release branches. Consider incrementing the patch version (e.g., `v1.0.1-prod`) instead.

---

## Part IV — Pipeline Internals: Build → Push → Deploy

This section provides a deep-dive explanation of each automated phase executed by the `deploy.yml` workflow after a tag push is detected.

### Phase 1: Build (GitHub Actions Cloud Runner)

The build phase executes entirely on GitHub's hosted infrastructure (`ubuntu-latest`), consuming zero resources from the target VPS.

| Step                   | Action                                         | Purpose                                                                                                               |
| ---------------------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **Checkout**           | `actions/checkout@v4`                          | Downloads the exact commit snapshot referenced by the pushed tag                                                      |
| **Env Injection**      | `printf "%s" "${{ secrets.ENV_FILE }}" > .env` | Writes the production `.env` file from GitHub Secrets to the runner workspace                                         |
| **Stack Setup**        | Language-specific actions                      | Installs the runtime (PHP, Node.js, Python, Go) and compiles dependencies/assets                                      |
| **Buildx Init**        | `docker/setup-buildx-action@v3`                | Activates BuildKit, enabling efficient layer caching and multi-platform builds                                        |
| **Username Normalize** | `tr '[:upper:]' '[:lower:]'`                   | Converts the GitHub actor name to lowercase (GHCR namespace requirement)                                              |
| **GHCR Login**         | `docker/login-action@v3`                       | Authenticates the runner against `ghcr.io` using the ephemeral `GITHUB_TOKEN`                                         |
| **Build & Push**       | `docker/build-push-action@v5`                  | Builds the container image from the `Dockerfile` and publishes it to GHCR with two tags: the version tag and `latest` |

**Image naming convention:**

```
ghcr.io/<github-actor-lowercase>/<repository-name>:<tag>
ghcr.io/<github-actor-lowercase>/<repository-name>:latest

Example:
ghcr.io/wildanfathirqinthara/my-app:v1.0.0-prod
ghcr.io/wildanfathirqinthara/my-app:latest
```

**Build caching:** The workflow leverages GitHub Actions cache (`type=gha`) for Docker layer caching. Subsequent builds that haven't changed a layer reuse the cached data, dramatically reducing build times (from ~5 minutes to under 60 seconds on cache hits).

---

### Phase 2: Push (GitHub Container Registry)

Once the build completes successfully, the runner pushes the compiled image to **GitHub Container Registry (GHCR)** — a private, access-controlled container registry hosted at `ghcr.io`.

- **Visibility:** Images inherit repository visibility by default. For private repositories, images are private and can only be pulled by authenticated users/tokens.
- **Retention:** Images remain in GHCR indefinitely until manually deleted. Old tags accumulate — consider implementing a retention policy via GitHub Actions or the GHCR UI.
- **Image Inspect:** After a push, you can verify the published image at:
  ```
  https://github.com/<owner>/<repository>/pkgs/container/<repository>
  ```

---

### Phase 3: Deploy (Target VPS via SSH)

After the image is published to GHCR, the runner establishes a secure SSH connection to the VPS and executes a sequential deployment script. Each sub-step is numbered in the workflow logs:

**[1/7] GHCR Authentication on Server**

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u "$ACTOR_LOW" --password-stdin
```

The server authenticates against GHCR using the Personal Access Token (`GHCR_TOKEN`). The `--password-stdin` flag prevents the token from appearing in the process list or shell history.

**[2/7] Image Pull**

```bash
docker pull ghcr.io/${ACTOR_LOW}/${APP_NAME}:${IMAGE_TAG}
```

Pulls only the new/changed layers from GHCR. Unchanged layers are served from Docker's local cache, minimizing bandwidth consumption on the VPS.

**[3/7] Container Stop & Remove**

```bash
docker stop "${APP_NAME}" || true
docker rm   "${APP_NAME}" || true
```

Gracefully terminates the SIGTERM → SIGKILL sequence on the running container. The `|| true` ensures the script does not fail on the first deployment when no prior container exists.

**[4/7] Network & Volume Provisioning**

```bash
docker network create "${NETWORK_NAME}" || true
docker volume create "${VOLUME_NAME}" || true
```

Creates the isolated bridge network and persistent storage volume if they don't already exist. These are **idempotent** operations — safe to run on every deployment without side effects or data loss.

**[5/7] Container Start**

```bash
docker run -d \
  --name "${APP_NAME}" \
  --network "${NETWORK_NAME}" \
  --restart unless-stopped \
  -v "${VOLUME_NAME}":/var/www/html/storage/app/public \
  "${FULL_IMAGE}"
```

Launches the new container in **detached daemon mode** (`-d`). The `--restart unless-stopped` policy ensures the container automatically restarts after server reboots or Docker daemon restarts, without requiring manual intervention.

**[6/7] Permission Fix**

```bash
sleep 5
docker exec "${APP_NAME}" chown -R www-data:www-data /var/www/html/storage || true
docker exec "${APP_NAME}" chmod -R 775 /var/www/html/storage || true
```

Docker's default volume mount behavior creates directories as `root:root` ownership. Web server processes (Nginx, PHP-FPM) running as `www-data` cannot write to these directories, causing HTTP 500 errors. This step corrects the ownership immediately after startup. The `sleep 5` provides a startup buffer.

**[7/7] Image Pruning**

```bash
docker image prune -af
```

Removes all **dangling images** (images with no tag, created by previous builds) and unreferenced intermediate layers. On a low-resource server with limited disk space, this step is essential to prevent disk exhaustion over time.

---

## Part V — Discord ChatOps Monitoring

The pipeline broadcasts two types of real-time notifications to your designated Discord channel via the `Ilshidur/action-discord@master` action.

### Success Notification

Fires when **all** pipeline steps complete with exit code `0`. The message contains:

| Field        | Value                                                                       |
| ------------ | --------------------------------------------------------------------------- |
| Status       | 🟢 ONLINE — Container is running                                            |
| Repository   | Full `owner/repo-name` path                                                 |
| Release Tag  | The exact tag that triggered the deploy (e.g., `v1.0.0-prod`)               |
| Triggered By | The GitHub username of the actor who pushed the tag                         |
| Commit       | Clickable link to the exact commit on GitHub                                |
| Run ID       | Clickable link to the Actions run log for audit purposes                    |
| Live URL     | `https://` + `DOMAIN_NAME` secret — direct link to the deployed application |

### Failure Notification

Fires when **any** pipeline step exits with a non-zero exit code (compilation error, Docker build failure, SSH connection timeout, container crash, etc.). The message contains:

| Field            | Value                                                                 |
| ---------------- | --------------------------------------------------------------------- |
| Status           | 🔴 FAILED — Rollback may be required                                  |
| Repository       | Full `owner/repo-name` path                                           |
| Release Tag      | The tag that triggered the failed deploy                              |
| Triggered By     | The GitHub username of the actor who pushed the tag                   |
| Diagnostics Link | Direct URL to the full GitHub Actions run log for error investigation |

### Interpreting Notification Status

| Discord Message          | Meaning                                 | Action Required                                                                                                         |
| ------------------------ | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| 🟢 DEPLOYMENT SUCCESSFUL | All steps passed; new container is live | Verify application via the live URL                                                                                     |
| 🔴 DEPLOYMENT FAILED     | One or more steps failed                | Click the diagnostics link, inspect the failed step's log output, fix the root cause, and re-trigger via a new tag push |

### Accessing GitHub Actions Run Logs

For detailed diagnostics beyond the Discord notification:

1. Navigate to your repository on GitHub.
2. Click the **Actions** tab.
3. Locate the failed run (marked with a red ✗ icon).
4. Click on the run title to expand the job view.
5. Click on any failed step (marked in red) to expand its log output.
6. Read the full terminal output to identify the specific error message.

### Common Failure Scenarios and Resolutions

| Symptom                                          | Root Cause                                                        | Resolution                                                          |
| ------------------------------------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------- |
| SSH connection timed out                         | `SERVER_IP` or `SERVER_USER` secret incorrect                     | Verify secrets; test manual SSH connection                          |
| `permission denied (publickey)`                  | `SERVER_SSH_KEY` incorrect or public key not in `authorized_keys` | Re-verify key setup in Part I, Section 1.3                          |
| `Error response from daemon: pull access denied` | `GHCR_TOKEN` expired or incorrect                                 | Regenerate the PAT and update the secret                            |
| Container exits immediately after start          | Application crashes on startup (check `APP_KEY`, `DB_*` env vars) | Run `docker logs <APP_NAME>` on the server; fix `.env` values       |
| HTTP 500 on first deployment                     | Storage directory permission error                                | Verify the `chown`/`chmod` step paths match your framework          |
| Disk full on VPS                                 | Old Docker images accumulated                                     | Run `docker image prune -af` and `docker system prune -af` manually |

---

## Troubleshooting Reference

### Useful Server-Side Diagnostic Commands

```bash
# View running containers
docker ps

# View all containers (including stopped)
docker ps -a

# View real-time logs from the application container
docker logs -f YOUR_APP_NAME

# View container resource consumption
docker stats YOUR_APP_NAME

# Inspect container details (network, volumes, env vars)
docker inspect YOUR_APP_NAME

# Execute a shell inside the running container
docker exec -it YOUR_APP_NAME /bin/sh

# Check available disk space
df -h

# Check current memory and SWAP usage
free -h

# Manually remove unused Docker resources
docker system prune -af --volumes
```

### SWAP Verification Commands

```bash
# Verify SWAP is active
swapon --show

# Monitor real-time memory usage
watch -n 1 free -h

# Check fstab persistence entry
grep swapfile /etc/fstab
```

### SSH Connectivity Test (from Local Machine)

```bash
# Test SSH connection with verbose output for debugging
ssh -vvv -i ~/.ssh/id_ed25519_deploy YOUR_USER@YOUR_SERVER_IP

# Test Docker access through SSH
ssh YOUR_USER@YOUR_SERVER_IP "docker ps"
```
