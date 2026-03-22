# CI/CD Pipeline Templates

Production-ready CI/CD pipeline templates for **GitHub Actions**, **GitLab CI**, and **Jenkins**. Covers Docker image builds, Kubernetes deployments, Terraform automation, security scanning, and auto-rollback. Based on real pipelines deploying 15 microservices to GKE.

---

## Problem Statement

Each microservice team creates pipelines from scratch — inconsistent build processes, no standardized security scanning, manual image tagging, and ad-hoc deployment scripts. Need reusable, production-grade pipeline templates that enforce best practices: automated testing, vulnerability scanning, semantic versioning, and GitOps-compatible deployment.

## Solution

A library of composable CI/CD templates for:
- Multi-stage Docker builds with layer caching
- Automated container image scanning (Trivy)
- Helm chart linting and packaging
- GitOps deployment (ArgoCD image updater or commit-based)
- Environment promotion (staging → production)

---

## Repository Structure

```
ci-cd-pipeline-templates/
├── gitlab-ci/
│   ├── .gitlab-ci.yml                    # Root pipeline (include templates)
│   ├── templates/
│   │   ├── docker-build.yml              # Multi-stage Docker build + push
│   │   ├── helm-deploy.yml               # Helm upgrade/install to K8s
│   │   ├── security-scan.yml             # Trivy container scanning
│   │   ├── test.yml                      # Unit/integration test stage
│   │   └── argocd-sync.yml              # ArgoCD app sync trigger
│   └── examples/
│       ├── backend-pipeline.yml          # Full backend CI/CD example
│       ├── frontend-pipeline.yml         # Frontend build + deploy
│       └── infrastructure-pipeline.yml   # Terraform plan/apply
│
├── github-actions/
│   ├── workflows/
│   │   ├── docker-build-push.yml         # Build → Scan → Push to registry
│   │   ├── helm-deploy.yml               # Helm deploy to GKE
│   │   ├── terraform-plan-apply.yml      # Infra provisioning pipeline
│   │   └── pr-checks.yml                 # Lint, test, security on PRs
│   └── composite-actions/
│       ├── docker-build/
│       │   └── action.yml                # Reusable Docker build action
│       ├── gke-auth/
│       │   └── action.yml                # GKE authentication action
│       └── helm-deploy/
│           └── action.yml                # Reusable Helm deploy action
│
├── dockerfiles/
│   ├── node-app.Dockerfile               # Multi-stage Node.js build
│   ├── python-app.Dockerfile             # Multi-stage Python build
│   └── nginx-spa.Dockerfile              # SPA with NGINX serve
│
├── docs/
│   ├── gitlab-ci-setup.md                # GitLab CI integration guide
│   ├── github-actions-setup.md           # GitHub Actions setup guide
│   └── argocd-integration.md             # GitOps deployment flow
│
└── README.md
```

---

## Pipeline Architecture

### GitLab CI Pipeline Flow

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Build   │ →  │   Test   │ →  │   Scan   │ →  │  Push    │ →  │  Deploy  │
│          │    │          │    │          │    │          │    │          │
│ Docker   │    │ Unit     │    │ Trivy    │    │ Artifact │    │ ArgoCD   │
│ multi-   │    │ tests    │    │ CVE scan │    │ Registry │    │ sync or  │
│ stage    │    │ lint     │    │ CRITICAL │    │ tag:     │    │ Helm     │
│ build    │    │ format   │    │ = fail   │    │ $CI_SHA  │    │ upgrade  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                                      │
                                                         ┌────────────┤
                                                         ▼            ▼
                                                    ┌─────────┐ ┌─────────┐
                                                    │ Staging │ │  Prod   │
                                                    │ (auto)  │ │(manual) │
                                                    └─────────┘ └─────────┘
```

### GitHub Actions Workflow

```
┌──────────────────────────────────────────────────────────────┐
│                    Pull Request                              │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Lint   │  │  Test   │  │  Build   │  │ Trivy Scan   │  │
│  │ (parallel)          │  │ (no push)│  │ (report PR)  │  │
│  └─────────┘  └─────────┘  └──────────┘  └──────────────┘  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    Merge to main                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │  Build   │→ │  Push    │→ │  Deploy  │→ │  Verify    │  │
│  │  + Tag   │  │  Registry│  │  Staging │  │  Health    │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │
│                                                   │         │
│                                          ┌────────▼──────┐  │
│                                          │ Deploy Prod   │  │
│                                          │ (manual gate) │  │
│                                          └───────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Key Features

- **Reusable Templates** — GitLab CI `include` and GitHub Actions composite actions for DRY pipelines
- **Multi-Stage Docker Builds** — Minimal production images; build dependencies excluded from runtime
- **Security Scanning** — Trivy container vulnerability scanning; fails on CRITICAL/HIGH CVEs
- **Artifact Registry** — Push to GCP Artifact Registry or Docker Hub with commit SHA tags
- **GitOps Compatible** — ArgoCD sync trigger or image tag update in Git for deployment
- **Environment Promotion** — Auto-deploy to staging; manual approval gate for production
- **Caching** — Docker layer caching and dependency caching for faster builds
- **Semantic Versioning** — Git tag-based versioning for release builds

---

## Tech Stack

| Component | Technology |
|---|---|
| **CI/CD** | GitLab CI/CD, GitHub Actions |
| **Container** | Docker (multi-stage builds) |
| **Registry** | GCP Artifact Registry, Docker Hub |
| **Security** | Trivy, Hadolint (Dockerfile lint) |
| **Deployment** | Helm v3, ArgoCD |
| **Target** | GKE, Kubernetes |

---

## Usage Example

### GitLab CI — Backend Service

```yaml
include:
  - local: templates/docker-build.yml
  - local: templates/security-scan.yml
  - local: templates/helm-deploy.yml

variables:
  APP_NAME: suidex-api-backend
  DOCKERFILE: Dockerfile
  HELM_CHART: apps/suidex-api-backend

stages:
  - build
  - scan
  - deploy

build:
  extends: .docker-build
  variables:
    REGISTRY: us-east4-docker.pkg.dev/suidex/suidex

scan:
  extends: .trivy-scan
  needs: [build]

deploy-staging:
  extends: .helm-deploy
  variables:
    ENVIRONMENT: staging
    NAMESPACE: suidex-api-backend
  when: on_success

deploy-production:
  extends: .helm-deploy
  variables:
    ENVIRONMENT: production
    NAMESPACE: suidex-api-backend
  when: manual
```

---

## Screenshots (Suggested)

- GitLab CI pipeline view showing all stages (build → test → scan → deploy)
- GitHub Actions workflow run with security scan results
- ArgoCD UI showing synced application after pipeline trigger
- Trivy scan report in merge request comment

---

## Author

**Sanket Raut** — DevOps Engineer  
[LinkedIn](https://linkedin.com/in/sanket-raut) · [Email](mailto:sanketraut.cloud@gmail.com)
