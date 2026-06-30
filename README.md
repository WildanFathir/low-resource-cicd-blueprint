<div align="center">

# ⚡ Low-Resource CI/CD Blueprint

### A Production-Grade, Cloud-Offloaded CI/CD Architecture for Constrained Server Environments

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-2088FF?logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![Docker](https://img.shields.io/badge/Docker-Container%20Runtime-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![GHCR](https://img.shields.io/badge/GHCR-Container%20Registry-181717?logo=github&logoColor=white)](https://ghcr.io)
[![Discord](https://img.shields.io/badge/ChatOps-Discord-5865F2?logo=discord&logoColor=white)](https://discord.com)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

**Born from academic research. Hardened for real-world production.**

_Originally designed and implemented as part of the undergraduate thesis:_
_"Perancangan dan Implementasi Arsitektur CI/CD Pada Lingkungan Low Resource Server Pada Proyek Aplikasi Web Presensi Cimalaka"_
_— Wildan Fathir Qinthara_

</div>

---

## 📖 Table of Contents

1. [Project Introduction & Core Concept](#-project-introduction--core-concept)
2. [The Offloaded Computation Strategy](#-the-offloaded-computation-strategy)
3. [Core Features Matrix](#-core-features-matrix)
4. [Repository Structure](#-repository-structure)
5. [Quick Start](#-quick-start)
6. [Documentation](#-documentation)
7. [Contributing & Open-Source Spirit](#-contributing--open-source-spirit-pull-requests)
8. [Contribution Ideas We Love](#-contribution-ideas-we-love)
9. [License](#-license)

---

## 🎯 Project Introduction & Core Concept

Modern CI/CD pipelines are almost universally designed with the assumption of **abundant compute resources** — multi-core build servers, 8 GB+ RAM pools, and high-bandwidth network links. For the vast majority of independent developers, student projects, startups, and small-to-medium enterprises operating on budget VPS tiers, this assumption is catastrophically wrong.

**This repository is an architectural blueprint** — a reusable, battle-tested CI/CD framework designed from the ground up for the **harshest constraint imaginable in production infrastructure: a single VPS with 1 vCPU and 1 GB of RAM.**

The framework was validated in the context of a real-world web attendance application (_Presensi Cimalaka_) as part of formal undergraduate research at Universitas Sebelas April (UNSAP). The result was a pipeline that achieved **fully automated, zero-downtime deployments** on a server configuration that most CI/CD documentation would consider inadequate for production use.

> **The central engineering challenge:** Running `composer install`, `npm run build`, and `docker build` simultaneously on a 1 GB RAM machine is a guaranteed path to an OOM (Out-Of-Memory) crash. The kernel kills processes indiscriminately. Deployments fail silently. Services go down.
>
> **This blueprint solves that problem permanently** — not by upgrading the server, but by rethinking _where_ each phase of the pipeline executes.

### Who This Is For

| Audience                      | Use Case                                                                         |
| ----------------------------- | -------------------------------------------------------------------------------- |
| **Independent developers**    | Deploying personal projects to budget VPS instances                              |
| **Startup engineering teams** | Cost-constrained production environments where server upgrades are not feasible  |
| **Academic researchers**      | A well-documented, peer-reviewed CI/CD implementation for study and extension    |
| **DevOps engineers**          | A portable, multi-stack template to adapt for client projects                    |
| **Open-source contributors**  | A living framework to extend with new stacks, security tooling, and integrations |

---

## 🧠 The Offloaded Computation Strategy

The fundamental innovation of this blueprint is a deliberate architectural pattern we call the **Offloaded Computation Strategy (OCS)**: the permanent and complete transfer of all computationally intensive pipeline phases away from the target server and onto GitHub's free cloud runner infrastructure.

```
┌─────────────────────────────────────────────────────────────────────┐
│              OFFLOADED COMPUTATION STRATEGY — OVERVIEW              │
├─────────────────────────────┬───────────────────────────────────────┤
│   GITHUB ACTIONS RUNNER     │         TARGET VPS                    │
│   (GitHub Cloud — Free)     │   (1 vCPU · 1 GB RAM — YOUR SERVER)  │
├─────────────────────────────┼───────────────────────────────────────┤
│ ✅ Source code checkout      │ ✅ docker login  (authentication)     │
│ ✅ PHP / Composer install    │ ✅ docker pull   (image download)     │
│ ✅ Node.js / npm install     │ ✅ docker stop   (graceful teardown)  │
│ ✅ npm run build (Vite/Mix)  │ ✅ docker run    (container start)    │
│ ✅ Docker image build        │ ✅ chown / chmod (permission fix)     │
│ ✅ GHCR image push           │ ✅ docker prune  (disk cleanup)       │
├─────────────────────────────┴───────────────────────────────────────┤
│  PEAK RAM CONSUMED ON VPS: ~180 MB (docker pull + run operations)   │
│  PEAK RAM CONSUMED ON RUNNER: ~1.4 GB (build phase) — NOT YOUR VPS │
└─────────────────────────────────────────────────────────────────────┘
```

### How It Works, End to End

```
Developer Workstation
        │
        │  git push origin v1.0.0-prod
        ▼
 GitHub Repository
        │
        │  Tag matches 'v*-prod' pattern → Workflow triggered
        ▼
 GitHub Actions Runner (ubuntu-latest — GitHub's infrastructure)
        │
        ├── [1] Checkout tagged commit snapshot
        ├── [2] Inject production .env from GitHub Secrets
        ├── [3] Install language runtime & compile assets
        │        (PHP/Composer, Node.js/Vite, Python/pip, Go build)
        ├── [4] Initialize Docker Buildx (BuildKit)
        ├── [5] Authenticate with ghcr.io
        └── [6] Build & push Docker image to GHCR
                       │
                       │  Image published → SSH tunnel opened
                       ▼
        Target VPS (YOUR low-resource server)
               │
               ├── [1/7] docker login ghcr.io
               ├── [2/7] docker pull <new image>
               ├── [3/7] docker stop + docker rm <old container>
               ├── [4/7] docker network create + volume create
               ├── [5/7] docker run <new container>
               ├── [6/7] chown + chmod (permission hardening)
               └── [7/7] docker image prune -af
                              │
                              ▼
                    Discord ChatOps Notification
                    (🟢 Success  |  🔴 Failure)
```

**The VPS never runs a compiler, never resolves a dependency tree, and never executes a multi-gigabyte build context.** Its sole runtime responsibility is container lifecycle management — operations with a predictable, minimal memory footprint that any 1 GB RAM machine can handle comfortably, especially when paired with the 2 GB SWAP buffer described in the [Usage Guide](USAGE-GUIDE.md).

---

## ✨ Core Features Matrix

### 🏗️ Offloaded Cloud Builds

- All CPU/RAM-intensive operations (dependency resolution, asset compilation, image layering) execute on **GitHub's free `ubuntu-latest` runners**, completely decoupled from your server's resource constraints.
- Supports **4 plug-and-play application stacks** via commented option blocks in `deploy.yml`: PHP/Laravel + Node.js/Vite, Python/FastAPI/Django, Go/Golang, and Pure Static Frontend (React/Vue/Angular/Next.js SSG).
- **BuildKit layer caching** via `type=gha` cache backend dramatically reduces subsequent build times — unchanged layers are served from cache rather than recompiled.

### 🐳 Immutable Runtime Environments via Docker

- Every deployment produces a **tagged, content-addressable Docker image** pushed to GitHub Container Registry (GHCR). The running container on your VPS always corresponds to an exact, auditable commit.
- The `--restart unless-stopped` container policy ensures **automatic recovery** from server reboots and Docker daemon restarts without manual intervention.
- **Named volumes** preserve persistent application data (uploaded files, database state) across container replacements, ensuring zero data loss on every redeployment.
- **Isolated bridge networks** enable multi-container topologies (app + database + reverse proxy) on the same low-resource host without port conflicts.

### 🗑️ Zero-Dangling Storage Optimization

- Every successful deployment concludes with an automatic `docker image prune -af` command executed on the VPS, **removing all dangling (untagged) image layers** left behind by previous builds.
- This prevents the most common silent failure mode on constrained servers: gradual disk space exhaustion leading to failed `docker pull` operations weeks after initial deployment.
- Combined with the dual-tagging strategy (`v1.0.0-prod` + `latest`), the registry and local host are always kept clean and predictable.

### 📡 Real-Time ChatOps Alerts via Discord

- A **Discord Incoming Webhook** integration broadcasts structured deployment status notifications directly to your designated monitoring channel after every pipeline run — no polling, no dashboard logins required.
- **Success notifications** include: repository name, release tag, triggering actor, clickable commit link, clickable Actions run link, and the live application URL.
- **Failure notifications** include: a direct diagnostics link to the full GitHub Actions run log, enabling immediate root-cause investigation without leaving your communication platform.
- The **failure condition** (`if: failure()`) is evaluated regardless of which step failed — SSH timeout, Docker build error, OOM event, or permission issue — ensuring no silent failures ever go undetected.

### 🔒 Secure Secrets Management

- All sensitive credentials (server IP, SSH private key, container registry token, domain name, Discord webhook, production `.env` content) are stored exclusively as **GitHub Repository Secrets**, encrypted at rest using libsodium sealed boxes.
- Secrets are **automatically masked** in all Actions log output — they appear as `***` and are never exposed in the run history.
- The pipeline uses **two distinct authentication tokens** for GHCR: the ephemeral `GITHUB_TOKEN` (runner-side build push) and a scoped PAT `GHCR_TOKEN` (server-side image pull), following the principle of least privilege.

### 🏷️ Immutable Tag-Based Release Strategy

- Production deployments are triggered **exclusively by Git tags** matching the `v*-prod` pattern. Branch pushes — including direct commits to `main` — **never** trigger a deployment, eliminating accidental production releases.
- Every deployment is permanently traceable to a specific commit snapshot via the Git tag, providing a built-in, human-readable deployment audit log.

---

## 📁 Repository Structure

```
low-resource-cicd-blueprint/
│
├── .github/
│   └── workflows/
│       └── deploy.yml          # Master CI/CD pipeline workflow
│
├── SECRETS.md                  # Complete secret configuration reference manual
├── USAGE-GUIDE.md              # Deep-dive server setup & deployment guide
└── README.md                   # This file
```

---

## 🚀 Quick Start

### Prerequisites

Before triggering your first deployment, complete the following:

**On your VPS (once, during initial setup):**

1. Allocate a **2 GB SWAP file** to prevent OOM crashes → [USAGE-GUIDE.md §1.1](USAGE-GUIDE.md#11-swap-memory-allocation-oom-prevention)
2. Install the **official Docker Engine** → [USAGE-GUIDE.md §1.2](USAGE-GUIDE.md#12-docker-engine-installation)
3. Configure **passwordless SSH + Docker group access** → [USAGE-GUIDE.md §1.3](USAGE-GUIDE.md#13-non-root-ssh-user-configuration)

**In your GitHub repository:** 4. Configure all **7 required Repository Secrets** → [SECRETS.md](SECRETS.md)

**In your application repository:** 5. Add a `Dockerfile` to the repository root → [USAGE-GUIDE.md §Part II](USAGE-GUIDE.md#part-ii--project-dockerfile-requirements) 6. Copy `deploy.yml` to `.github/workflows/deploy.yml` and uncomment the stack block matching your framework.

### Trigger Your First Deployment

```bash
# 1. Commit and push your application code
git add .
git commit -m "feat: initial production release"
git push origin main

# 2. Create and push a production release tag
git tag v1.0.0-prod
git push origin v1.0.0-prod

# 3. Monitor the pipeline
# Navigate to: https://github.com/<owner>/<repo>/actions
# A "Build & Push & Deploy Application" run will appear immediately.

# 4. Check your Discord channel for the deployment result notification.
```

---

## 📚 Documentation

| Document                                                     | Purpose                                                                                                                      |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| [USAGE-GUIDE.md](USAGE-GUIDE.md)                             | Complete server hardening guide, Dockerfile patterns, deployment triggers, pipeline deep-dive, and troubleshooting reference |
| [SECRETS.md](SECRETS.md)                                     | Step-by-step retrieval and configuration guide for all 7 GitHub Actions secrets                                              |
| [.github/workflows/deploy.yml](.github/workflows/deploy.yml) | The annotated master pipeline workflow file                                                                                  |

---

## 🤝 Contributing & Open-Source Spirit (Pull Requests)

### The Philosophy

This project began as a formal academic research artifact — a thesis submitted in partial fulfillment of a Bachelor's degree in Informatics at Universitas Sebelas April. But **research should not die in a PDF**.

The spirit behind this blueprint is borrowed from the same ethos that Linus Torvalds articulated when he posted a note to the Minix newsgroup in 1991: _"I'm doing a (free) operating system (just a hobby, won't be big and professional like gnu)..."_ — and then proceeded to change computing forever, not because he built it alone, but because he opened it to the world.

This CI/CD framework was designed to solve a real problem that affects thousands of developers daily: the gap between what modern DevOps tooling assumes about infrastructure and what independent developers and small teams can actually afford. **That gap is worth closing, and it cannot be closed by a single researcher working alone.**

Every improvement — whether it's a new stack template, a security enhancement, a documentation correction, or a translated guide — makes this blueprint more useful to a developer somewhere who is wrestling with a 1 GB RAM VPS at 2 AM trying to get their first production deployment to work.

**Pull requests are not just welcome here — they are the entire point.**

---

### 📋 Pull Request Workflow Guide

Follow these GitOps conventions to submit a contribution cleanly and efficiently.

#### Step 1 — Fork the Repository

Create your personal copy of the repository under your GitHub account by clicking the **Fork** button at the top-right of the repository page on GitHub.

#### Step 2 — Clone Your Fork Locally

```bash
# Clone your forked copy to your local development machine.
# Replace YOUR_USERNAME with your actual GitHub username.
git clone https://github.com/YOUR_USERNAME/low-resource-cicd-blueprint.git

# Navigate into the repository directory.
cd low-resource-cicd-blueprint
```

#### Step 3 — Add the Upstream Remote

Configure the original repository as a remote named `upstream`. This allows you to pull in future changes from the source project to keep your fork synchronized.

```bash
git remote add upstream https://github.com/wildanfathirqinthara/low-resource-cicd-blueprint.git

# Verify both remotes are configured correctly.
git remote -v
# Expected output:
# origin    https://github.com/YOUR_USERNAME/low-resource-cicd-blueprint.git (fetch)
# origin    https://github.com/YOUR_USERNAME/low-resource-cicd-blueprint.git (push)
# upstream  https://github.com/wildanfathirqinthara/low-resource-cicd-blueprint.git (fetch)
# upstream  https://github.com/wildanfathirqinthara/low-resource-cicd-blueprint.git (push)
```

#### Step 4 — Sync Your Fork with Upstream (Before Every New Contribution)

Always start from an up-to-date base to minimize merge conflicts.

```bash
# Fetch the latest changes from the upstream repository.
git fetch upstream

# Switch to your local main branch.
git checkout main

# Merge upstream changes into your local main branch.
git merge upstream/main

# Push the synchronized state to your fork's remote.
git push origin main
```

#### Step 5 — Create an Isolated Feature Branch

**Never commit directly to `main`.** Create a descriptive branch that clearly communicates the scope of your contribution.

```bash
# Branch naming convention:
#   feature/   — new capabilities or stack templates
#   fix/        — bug corrections or broken configurations
#   docs/       — documentation improvements or translations
#   security/   — DevSecOps, SAST, or hardening contributions
#   chore/      — maintenance tasks (dependency bumps, refactoring)

# Examples:
git checkout -b feature/python-fastapi-stack-template
git checkout -b docs/bahasa-indonesia-translation
git checkout -b security/trivy-sast-scanning-step
git checkout -b fix/go-binary-volume-mount-path
```

#### Step 6 — Make Your Changes

Implement your contribution. Keep changes focused and atomic — one logical concern per pull request. This makes review faster and merging safer.

#### Step 7 — Commit Using Semantic Conventions

This project follows the [Conventional Commits](https://www.conventionalcommits.org/) specification. Well-formed commit messages make the changelog self-documenting and review history immediately legible.

```bash
# Stage your changes.
git add .

# Commit with a semantic message following the convention:
# <type>(optional scope): <imperative description>
#
# Types:
#   feat     — a new feature or capability
#   fix      — a bug fix
#   docs     — documentation-only changes
#   refactor — code restructuring without behavior change
#   security — security-related improvements
#   chore    — maintenance, dependency updates

# Examples:
git commit -m "feat(workflow): add Python/FastAPI plug-and-play stack option"
git commit -m "docs(readme): add Bahasa Indonesia translation section"
git commit -m "fix(deploy): correct Go container volume mount path"
git commit -m "security(workflow): integrate Trivy SAST image scanning step"
git commit -m "chore(actions): bump docker/build-push-action from v5 to v6"
```

#### Step 8 — Push Your Branch to Your Fork

```bash
# Push the feature branch to your forked remote repository.
git push origin feature/python-fastapi-stack-template
```

#### Step 9 — Open a Formal Pull Request on GitHub

1. Navigate to your fork on GitHub. A yellow banner will appear: **"Compare & pull request"** — click it.
2. Ensure the **base repository** is `wildanfathirqinthara/low-resource-cicd-blueprint` and the **base branch** is `main`.
3. Fill in the PR description using the following template:

```markdown
## Summary

<!-- A clear, concise description of what this PR adds, changes, or fixes. -->

## Motivation

<!-- Why is this change needed? What problem does it solve?
     Link any related issues: Closes #123 -->

## Changes Made

<!-- List the specific files and changes: -->

- `deploy.yml`: Added commented Python/FastAPI stack option block
- `USAGE-GUIDE.md`: Added FastAPI Dockerfile example section

## Stack / Environment Tested

<!-- Describe the environment where you validated this change. -->

- OS: Ubuntu 22.04 LTS
- Docker: 26.1.4
- Tested with: FastAPI 0.111 + Uvicorn 0.30

## Checklist

- [ ] My changes follow the existing comment style and formatting conventions
- [ ] I have tested this configuration end-to-end in a real deployment environment
- [ ] I have updated the relevant documentation (USAGE-GUIDE.md / SECRETS.md) if applicable
- [ ] My commit messages follow the Conventional Commits specification
- [ ] I have not introduced any hardcoded values (IPs, usernames, tokens) into workflow files
```

4. Click **Create pull request** to submit.

A maintainer will review your contribution, provide feedback if needed, and merge it upon approval. Thank you for your patience — **every PR is read and valued**.

---

## 💡 Contribution Ideas We Love

Not sure where to start? Here are the contribution areas that would have the highest impact on the community of developers this project serves:

---

### 🧪 Lightweight Unit Testing Hooks

**The challenge:** Running `phpunit`, `jest`, or `pytest` as a CI gate is standard practice — but naively adding a full test suite runner to the pipeline requires careful consideration on constrained runners.

**What we're looking for:**

- Integration of **PHPUnit** (Laravel/Symfony) or **Pest PHP** as an optional pre-build gate step in `deploy.yml`, scoped only to unit tests (not integration/browser tests that require a running database).
- Integration of **Jest** or **Vitest** for JavaScript unit testing, with explicit configuration to exclude tests requiring a DOM environment that would inflate runner time.
- A clear `# OPTIONAL: Uncomment to enable unit test gate` comment pattern consistent with the multi-stack option blocks.

The key constraint is preserving the pipeline's **fast, predictable execution time**. Test steps should be parallelizable and should fail fast on the first failure rather than waiting for the full suite.

---

### 🛡️ DevSecOps / SAST Scanning Integration

**The challenge:** Security scanning is often treated as an enterprise-only concern. This project aims to democratize it for budget infrastructure projects.

**What we're looking for:**

- Integration of **[Trivy](https://github.com/aquasecurity/trivy)** (`aquasecurity/trivy-action`) as a post-build image vulnerability scanner. The scan should target the freshly built GHCR image and report `CRITICAL` and `HIGH` severity findings.
- Integration of **[Snyk](https://snyk.io/)** or **[OWASP Dependency-Check](https://jeremylong.github.io/DependencyCheck/)** for dependency manifest scanning (composer.lock, package-lock.json, requirements.txt).
- Optional: Integration of **[Semgrep](https://semgrep.dev/)** for lightweight SAST static analysis of application source code during the build phase.
- All security steps should follow the same commented opt-in pattern as the stack blocks — **no mandatory security gates that would break existing users' pipelines** without their explicit opt-in.

---

### 🔌 Plug-and-Play Multi-Fleet Stack Templates

**The challenge:** This blueprint currently ships with PHP/Laravel as the active default stack. The alternative stacks (Python, Go, Static Frontend) exist as commented blocks but lack dedicated, tested Dockerfile examples and documentation sections.

**What we're looking for:**

- **Python/FastAPI or Django:** A complete, production-tested `Dockerfile` using multi-stage builds with `python:3.11-slim`, a validated `requirements.txt` install pattern, and a corresponding `USAGE-GUIDE.md` section.
- **Go/Golang:** A validated multi-stage `Dockerfile` with `golang:1.22-alpine` compiler stage and `alpine:latest` minimal runtime, including the correct `CGO_ENABLED=0` cross-compilation flags.
- **Rust:** A multi-stage Rust `Dockerfile` (highly requested from the community) using `rust:alpine` builder and `alpine:latest` runtime — producing a statically linked binary with minimal image footprint.
- **Node.js/Express or NestJS:** A production-grade `Dockerfile` for Node.js API servers (distinct from the existing Static Frontend option).
- For each stack: document the correct volume mount paths (replacing the Laravel-specific `/var/www/html/storage` path) and the appropriate `chown` user:group for the permission fix step.

---

### 🌐 Multi-Stage Environment Pipelines (Staging → Production)

**The challenge:** This blueprint currently implements a single-environment pipeline (production only). Many projects require a staging environment for pre-release validation.

**What we're looking for:**

- A `deploy-staging.yml` workflow template that triggers on `v*-staging` tags, deploys to a separate staging server (`SERVER_IP_STAGING`, `SERVER_USER_STAGING`), and sends a distinct Discord notification with a staging URL.
- Documentation on using **GitHub Environment Secrets** (scoped to `production` and `staging` environments) instead of flat Repository Secrets for improved secret isolation.
- An optional rollback workflow (`rollback.yml`) triggered manually via `workflow_dispatch`, which re-deploys the previous tagged image without a full rebuild.

---

### 🌍 Documentation Translations

**The challenge:** This project originated in an Indonesian academic context but is useful globally. Language barriers should not prevent a developer in Vietnam, Brazil, or Morocco from understanding and using this framework.

**What we're looking for:**

- Translations of `README.md`, `USAGE-GUIDE.md`, and `SECRETS.md` into:
  - **Bahasa Indonesia** (the author's native language — an authoritative translation would be particularly valuable)
  - **Portuguese (Brazil)**
  - **Spanish**
  - **Vietnamese**
  - **Arabic**
  - Any other language where this tool would reach an underserved developer community
- Translations should be placed in a `docs/` directory with ISO 639-1 language codes as subdirectory names (e.g., `docs/id/README.md`, `docs/pt-BR/README.md`).

---

### 📊 Observability & Monitoring Integrations

**What we're looking for:**

- Integration of **[Uptime Kuma](https://github.com/louislam/uptime-kuma)** deployment status webhooks alongside (or as an alternative to) Discord notifications.
- A post-deployment health check step that performs an HTTP `curl` against `https://${{ secrets.DOMAIN_NAME }}/health` and fails the pipeline (triggering the failure Discord alert) if the endpoint returns a non-2xx status code.
- Integration with **[ntfy.sh](https://ntfy.sh/)** as a lightweight, self-hostable alternative notification backend for developers who do not use Discord.

---

## 📄 License

This project is released under the **MIT License**. See [LICENSE](LICENSE) for the full license text.

You are free to use, copy, modify, merge, publish, distribute, sublicense, and sell copies of this software, provided the original copyright notice is retained. This framework is given to the community freely and in good faith.

---

<div align="center">

**Built with academic rigor. Shared with open-source spirit.**

_Wildan Fathir Qinthara_

---

_"The Linux philosophy is 'laugh in the face of danger'. Oops. Wrong one. 'Do it yourself'. That's it."_
_— Linus Torvalds_

_Every great open-source project started as someone's personal problem._
_This one started on a 1 GB RAM VPS that kept running out of memory._

⭐ If this blueprint saved your deployment — consider starring the repository.

</div>
