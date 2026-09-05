# ADR-001: Use k3s for Local Kubernetes

## Status
Accepted

## Context
This project requires a Kubernetes cluster to run on local hardware (VirtualBox VMs on
a single physical machine), with no cloud budget assumed. Three common local-Kubernetes
options were considered:

- **kind** (Kubernetes-in-Docker): runs cluster nodes as Docker containers. Popular for
  CI pipelines and ephemeral test clusters.
- **minikube**: a single-node (or limited multi-node) local cluster, typically run inside
  a VM or container, aimed at individual developer sandboxes.
- **k3s**: a certified, lightweight Kubernetes distribution packaged as a single binary,
  designed to run directly on a VM or bare metal (including multi-node clusters), commonly
  used in edge computing, IoT, and resource-constrained production environments.

## Decision
Use **k3s**, installed directly on Ubuntu Server VMs (the same VM skillset already
established in prior project work), rather than kind or minikube.

## Reasoning
1. **Runs as a real system service.** k3s installs as a systemd-managed service on a
   real Linux host, which matches how a platform engineer would actually operate a
   cluster node (log inspection via `journalctl`, service management via `systemctl`),
   rather than abstracting the node away inside a container (kind) or a throwaway VM
   wrapper (minikube).
2. **Multi-node is straightforward and realistic.** k3s supports adding additional VMs
   as worker nodes with a single join command. This allows the architecture to evolve
   toward a more realistic multi-node topology later without switching tools.
3. **It is a real production distribution**, not only a dev tool. Organizations run k3s
   in production for edge/on-prem/resource-constrained clusters. This means the skills
   transfer directly, whereas kind is explicitly documented as not intended for
   production use.
4. **Fits the existing skillset.** Prior experience with systemd, static IPs, and
   Ubuntu Server administration transfers directly to operating a k3s node, whereas
   kind's Docker-in-Docker model would partially bypass that experience.

## Alternatives Rejected
- **kind**: excellent for CI/ephemeral test clusters, but the "nodes are containers"
  model is a poor match for a project meant to simulate operating real infrastructure.
- **minikube**: simpler onboarding, but weaker multi-node story and less production
  relevance than k3s for this project's goals.

## Consequences
- We give up some of the very fast startup/teardown convenience of kind for CI-style
  throwaway clusters. This is an acceptable trade-off since this cluster is meant to be
  long-lived, not ephemeral.
- We take on responsibility for basic node-level operations (OS updates, resource
  monitoring) that kind/minikube would abstract away — this is treated as a feature of
  the project (more realistic operational surface), not a downside.
- Full production Kubernetes (e.g., managed EKS/GKE/AKS) differs from k3s in several
  ways we will call out explicitly as they become relevant (e.g., k3s ships SQLite by
  default instead of etcd for the control plane datastore, uses Traefik as a default
  ingress controller, and bundles some components differently). These differences will
  be noted at the point they matter rather than assumed away.
