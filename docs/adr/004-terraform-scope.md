# ADR-004: Scope Terraform to Repository/Namespace Provisioning, Not VM Provisioning

## Status
Accepted

## Context
Terraform is an Infrastructure-as-Code tool most commonly associated with provisioning
cloud infrastructure (VPCs, VMs, managed Kubernetes clusters, etc.) via cloud provider
APIs. This project runs on local VirtualBox VMs rather than cloud infrastructure, which
raises the question of what, if anything, Terraform should manage here.

VirtualBox VM provisioning is technically possible via community Terraform providers,
but this is a narrow, less-maintained corner of the Terraform ecosystem, and doing it
well typically involves pairing Terraform with a tool like Vagrant or Packer — introducing
substantial tooling overhead for a local lab environment where VMs are created
infrequently and manual creation is well understood already (per prior project
experience).

## Decision
Terraform will **not** provision the VirtualBox VMs or the k3s installation itself.
Those remain manual/scripted (Bash) steps, documented clearly as "local lab setup."

Terraform **will** be used, starting in Phase 11, to provision things that map directly
to how it's used in real cloud-native platform teams even in a local context:
- GitHub repository creation (via the GitHub Terraform provider) — since Backstage's
  Software Templates will call Terraform (or an equivalent) to create new service
  repos in a controlled, auditable way, rather than shelling out to `git init` +
  manual GitHub UI clicks.
- Kubernetes namespace and RBAC resource provisioning (via the Kubernetes Terraform
  provider) — since this is a common real-world use of Terraform even against
  self-managed clusters.

## Reasoning
1. **Matches real-world division of labor.** In real companies, Terraform provisions
   cloud resources and Kubernetes-API-level resources; VM base images are typically
   handled by separate tooling (Packer, cloud-init, or the cloud provider directly) —
   so scoping Terraform away from "create the VM" is actually the more realistic
   choice, not a simplification.
2. **Avoids a low-value, high-friction detour.** Forcing Terraform to manage VirtualBox
   VMs would cost significant setup time for a capability (VM creation) already
   demonstrated in prior project work, without teaching a new transferable concept.
3. **Demonstrates Terraform where it actually adds value in this project**: as an
   automation primitive that Backstage's scaffolder invokes on a developer's behalf,
   which is a realistic and resume-honest use of "Infrastructure as Code integrated
   into a self-service platform."

## Alternatives Rejected
- **No Terraform at all**: would remove a commonly expected platform-engineering skill
  from the project entirely.
- **Terraform for VM provisioning via VirtualBox provider**: rejected due to poor
  effort-to-learning ratio for this project's goals (see Reasoning #2).

## Consequences
- The resume/portfolio claim regarding Terraform must be scoped honestly: "Terraform
  used for GitHub repository and Kubernetes namespace/RBAC provisioning, invoked as
  part of the self-service scaffolding flow" — not "Terraform used to provision cluster
  infrastructure," which would be inaccurate for this project.
- VM/k3s setup steps (Phase 1) will be documented as manual/Bash-scripted lab setup,
  explicitly labeled as such so it isn't confused with the Terraform-managed layer.
