# Requirements

## Problem Statement
Developers should be able to create and deploy a standardized application/service
through a self-service interface, without requiring a platform/DevOps engineer to
manually configure repositories, CI/CD, container builds, Kubernetes deployment,
documentation, or monitoring for each new service.

## Functional Requirements

| ID    | Requirement                                                                                          | Target Phase |
|-------|-------------------------------------------------------------------------------------------------------|--------------|
| FR-1  | A developer can deploy a containerized application to a local Kubernetes cluster                     | 3            |
| FR-2  | Deployment configuration is reusable across services via a parameterized Helm chart                  | 4            |
| FR-3  | Pushing code triggers automated lint, test, security scan, image build, and image push                | 5            |
| FR-4  | Built images are stored in a container registry and are pullable by the cluster                      | 6            |
| FR-5  | Deployment to the cluster happens via Git-based reconciliation (GitOps), not direct kubectl/CI deploys | 7            |
| FR-6  | A developer portal displays a catalog of services with owner, repo link, and docs                     | 8            |
| FR-7  | A developer can self-service create a new, fully-scaffolded service via a template                    | 9            |
| FR-8  | A service can request a provisioned database as part of scaffolding                                    | 10           |
| FR-9  | Secrets are not stored in plaintext in Git; RBAC restricts access appropriately                        | 11           |
| FR-10 | Metrics (CPU, memory, pod health, request rate, error rate, latency) are visible per service            | 12           |
| FR-11 | The portal shows live deployment status and version per service                                         | 12           |
| FR-12 | Controlled rollback of a bad deployment is possible and documented                                       | 13           |
| FR-13 | The platform survives and recovers from at least: pod kill, bad deploy, high load                        | 14           |
| FR-14 | Basic policy/security controls are enforced (non-root containers, network policies at minimum)            | 15           |
| FR-15 | At least 2 independent services are demonstrated using the same golden path                              | 9 (ongoing)  |

## Non-Functional Requirements

| ID     | Requirement                                                                                   |
|--------|------------------------------------------------------------------------------------------------|
| NFR-1  | Entire stack runs on local hardware (VirtualBox VMs); no paid cloud services required           |
| NFR-2  | Every architectural decision with more than one reasonable option is documented as an ADR       |
| NFR-3  | Every phase is independently verifiable (explicit verification steps, not "trust it works")     |
| NFR-4  | Documentation is written as the project is built, not reconstructed afterward                   |
| NFR-5  | Every resume/portfolio claim must map to something actually implemented and verifiable in-repo  |

## Explicit Non-Goals (for this project's scope)
- Multi-region / multi-cluster deployment
- Full production-grade high availability (e.g., etcd HA, multi-master control plane)
- Cloud provider integration (AWS/GCP/Azure) — explicitly a local lab exercise
- Distributed tracing (OpenTelemetry) — noted as a valid future extension, not required
- Full policy-as-code suite (OPA/Kyverno) beyond basic NetworkPolicy — stretch goal only if
  time remains after Phase 16

## Definition of Done (MVP)
FR-1 through FR-7 implemented and verified, with documentation for setup and developer
usage, and at least one service deployed end-to-end through the full pipeline.

## Definition of Done (Strong Portfolio Version)
All functional requirements (FR-1–FR-15) implemented and verified, with incident
documentation for at least one deliberately induced failure scenario (FR-13).
