# ADR-005: 8 GB RAM Hardware Ceiling Drives Single-Node, Incremental Component Introduction

## Status
Accepted

## Context
The development machine has 8 GB of total RAM and runs Windows as the host OS, with a
single Ubuntu Server VM (running k3s) as the guest. Windows itself, background
processes, and the hypervisor all need headroom. This is a materially tighter
constraint than a typical "local lab" assumption of 16–32 GB, and it must be treated as
a hard ceiling, not a soft preference — components that are normally run simultaneously
in a full local Kubernetes platform stack (Kubernetes control plane, Argo CD,
Prometheus, Grafana, Backstage, PostgreSQL) cannot all run at once here without either
starving the VM or making Windows unusable.

## Decision
1. **VM allocation**: 3 GB RAM / 2 CPU cores / 40 GB disk for the k3s VM (rationale
   below), leaving roughly 5 GB + remaining CPU for Windows and the hypervisor.
2. **Single-node k3s cluster** — no additional worker-node VM will be introduced unless
   a specific later learning objective genuinely requires observing multi-node
   scheduling behavior (and if so, it will be spun up temporarily and torn down after).
3. **Strictly incremental component introduction** — each phase installs only what that
   phase needs. Nothing from a later phase (Argo CD, Backstage, Prometheus, Postgres) is
   installed early "to save time later." This directly extends the phase ordering
   already set in ADR-003 (Backstage sequencing) to every other component.
4. **Resource requests/limits on every workload are set deliberately small**
   (e.g., tens of MB request, low CPU millicores) and reviewed at the point each
   workload is created — not left at defaults.
5. **1 replica by default** for all workloads, with replica counts >1 introduced only
   at the specific point a phase's learning objective requires observing that behavior
   (e.g., demonstrating rolling updates or load distribution).
6. **Components that are memory-heavy and not currently the focus of a phase are
   stopped, not left running.** For example, once Argo CD is understood and phase-goal
   evidence is collected, it can be scaled down (`kubectl scale deploy ... --replicas=0`
   in its namespace, or `systemctl stop k3s` entirely between working sessions) rather
   than left resident to "save re-setup time."
7. **Before introducing any new resource-heavy component**, we will explicitly state:
   why it consumes resources, whether it's necessary at that phase, whether it can be
   stopped when idle, and what lighter alternative exists if one does.

## Reasoning
A single k3s control plane alone uses a meaningful chunk of RAM (roughly 500 MB–1 GB
once steady-state, more under load) before any workload is scheduled. Stacking Argo CD
(has its own API server, repo-server, application controller, redis — each a separate
pod), Prometheus (in-memory time-series storage, which is the single most RAM-hungry
component per unit of "stuff monitored"), Grafana, and Backstage (a full Node.js
backend + bundled Postgres-backed catalog, arguably the heaviest single component in
this entire stack) simultaneously in a 3 GB VM would either cause pods to be OOMKilled
or push swap usage on the host, degrading Windows responsiveness — the opposite of the
project's own principle of a "stable laptop." Sequencing installation/use to match the
current learning phase — and stopping what isn't currently needed — keeps the working
set small at any given moment while still letting every phase be demonstrated and
evidenced individually.

## Alternatives Rejected
- **Allocate more RAM to the VM (e.g., 5–6 GB) and run everything concurrently**:
  rejected — would leave Windows with too little headroom given the 8 GB total, and
  contradicts the explicit hardware constraint.
- **Use a remote/cloud VM instead of local**: rejected per project principle of
  building locally before introducing cloud infrastructure, and not necessary — every
  phase's learning objective is achievable at this scale.
- **Run every component permanently at minimal replica/resource settings rather than
  stopping unused ones**: rejected — even minimal-footprint versions of 5+ components
  running concurrently still adds up meaningfully on a 3 GB VM; stopping what isn't the
  current focus is more effective than shrinking everything uniformly.

## Consequences
- Some phases will involve explicitly starting a previous phase's component back up to
  demonstrate integration (e.g., turning Argo CD back on in Phase 8+ to show Backstage
  displaying its status) — this will be called out as a deliberate "bring this back
  online" step, not treated as a surprise.
- Portfolio documentation must describe the platform's steady-state resource footprint
  honestly (what runs concurrently vs. what is toggled on/off), since a real production
  platform would run all of this concurrently on dedicated infrastructure — this
  difference will be noted explicitly in the final documentation (Phase 16) as one of
  the local-lab-vs-production distinctions.
- We will check `free -h` / `htop` / `df -h` (VM side) and Windows Task Manager (host
  side) before and after introducing each new component, and record the delta as part
  of that phase's evidence.
