# Internal Developer Platform (IDP)

A self-service Internal Developer Platform, built as a portfolio project, that lets a
developer scaffold, deploy, and observe a standardized service without manually
configuring repositories, CI/CD, Kubernetes, or monitoring.

> **Status:** In progress — Phase 0 (Architecture & Requirements) complete.
> See [docs/requirements.md](docs/requirements.md) for the full functional scope and
> [docs/adr/](docs/adr/) for architectural decisions.

## What This Project Demonstrates

```
Developer
   │
   ▼
Backstage Portal (Software Catalog + Software Templates + TechDocs)
   │
   ▼
New GitHub Repository (app skeleton, Dockerfile, CI workflow, Helm chart, docs)
   │
   ▼
GitHub Actions (CI): lint → test → security scan → build image → push
   │
   ▼
Container Registry (GHCR)
   │
   ▼
GitOps Repo (idp-gitops) ← CI updates image tag here
   │
   ▼
Argo CD (watches GitOps repo, reconciles cluster state)
   │
   ▼
Kubernetes (k3s)
   │
   ▼
Application Pods
   │
   ▼
Prometheus → Grafana → Alertmanager
   │
   ▼
Backstage displays live status back to the developer
```

## Why This Exists

In most engineering organizations past a certain size, creating a new service touches
CI conventions, Docker/base-image standards, Kubernetes/Helm conventions, secrets
handling, and monitoring setup — tribal knowledge usually held by a platform/DevOps
team. This project builds a "golden path": a standardized, self-service way for a
developer to create a new service that automatically follows the organization's best
practices, rather than requiring a platform engineer to hand-configure each one.

Full problem statement and architecture reasoning: [docs/requirements.md](docs/requirements.md).

## Hardware Constraint

Built on an 8 GB RAM development laptop (Windows host, Ubuntu Server VM guest running
k3s). This is a hard constraint, not a soft preference — see
[ADR-005](docs/adr/005-hardware-constraint-8gb-ram.md) for how it shapes VM sizing,
single-node-only clustering, incremental component rollout, and the practice of
stopping components that aren't the current phase's focus rather than running the full
stack concurrently.

## Local Lab vs. Production — An Honest Disclaimer

This project runs entirely on local VirtualBox VMs, not cloud infrastructure. Where a
real company would use a managed Kubernetes service, cloud IAM, or multi-region
infrastructure, this project uses k3s, local RBAC, and a single-cluster setup instead.
Every place this matters is called out explicitly in the relevant ADR or phase
documentation — the goal is to demonstrate the same engineering concepts on realistic
lab infrastructure, not to claim the lab is production-equivalent.

## Repository Structure

| Path                    | Purpose                                                               |
|--------------------------|------------------------------------------------------------------------|
| `docs/`                 | Requirements, ADRs, architecture diagrams, guides, incident reports    |
| `infrastructure/terraform/` | Terraform: GitHub repo + Kubernetes namespace/RBAC provisioning (Phase 11) |
| `kubernetes/manifests/` | Raw Kubernetes manifests (Phase 3, pre-Helm)                          |
| `helm/service-chart/`   | Generic, reusable Helm chart used by all scaffolded services (Phase 4) |
| `platform/backstage/`   | Backstage developer portal instance (Phase 8)                        |
| `templates/service-template/` | Backstage Software Template used for golden-path scaffolding (Phase 9) |
| `monitoring/`           | Prometheus/Grafana configuration (Phase 12)                           |
| `scripts/`              | Bootstrap and helper scripts                                          |
| `services/`             | Local references to scaffolded service repos (each service is its own repo) |

Related repositories (created later):
- `idp-gitops` — GitOps desired-state repo watched by Argo CD (Phase 7)
- Individual service repos (e.g., `payment-api`, `user-api`) — created via the golden
  path starting Phase 9

## Build Log / Phases

See [docs/requirements.md](docs/requirements.md) for the functional requirements each
phase satisfies. Phase-by-phase documentation lives under `docs/` as it's produced.

- [x] Phase 0 — Architecture and requirements
- [x] Phase 1 — Local Kubernetes foundation ([details](docs/phase-1-k3s-foundation.md))
- [x] Phase 2 — Containerized application ([details](docs/phase-2-containerized-application.md))
- [ ] Phase 3 — Kubernetes deployment
- [ ] Phase 4 — Helm-based reusable deployment
- [ ] Phase 5 — CI pipeline
- [ ] Phase 6 — Container registry
- [ ] Phase 7 — GitOps / Argo CD
- [ ] Phase 8 — Backstage developer portal
- [ ] Phase 9 — Golden Path / service templates
- [ ] Phase 10 — Database provisioning
- [ ] Phase 11 — Secrets and RBAC
- [ ] Phase 12 — Observability
- [ ] Phase 13 — Deployment strategies and rollback
- [ ] Phase 14 — Failure testing / self-healing
- [ ] Phase 15 — Security / policy
- [ ] Phase 16 — Documentation and portfolio presentation
