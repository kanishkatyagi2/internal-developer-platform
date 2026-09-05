# ADR-002: Two-Repository GitOps Model (App Repo vs. GitOps Repo)

## Status
Accepted

## Context
GitOps requires a Git repository to act as the single source of truth for desired
cluster state, which a tool such as Argo CD continuously reconciles against. A common
early design question is whether the Kubernetes manifests/Helm values for a service
should live inside the same repository as the application code ("single-repo") or in a
separate, dedicated repository ("two-repo" / "app repo + config repo").

## Decision
Use a **two-repository model**: each service has an application repository (code,
Dockerfile, CI workflow) and there is a single separate `idp-gitops` repository holding
the desired-state manifests/Helm values for all services and environments, which Argo CD
watches directly. CI updates the GitOps repo (e.g., bumping an image tag) as its final
step; it never deploys directly to the cluster.

## Reasoning
1. **Separates "what changed in code" from "what is currently deployed."** A commit to
   an app repo represents a code change. A commit to the GitOps repo represents a
   deployment event. Conflating these makes it hard to answer "what is running in
   production right now?" by looking at repo state alone.
2. **Avoids CI-triggered infinite loops.** If CI both builds an image from the app repo
   and also modifies deployment manifests in that same repo, a naive setup can trigger
   the pipeline to fire on its own commits. A separate repo, updated by a scoped
   "commit-back" step, keeps these concerns cleanly separated.
3. **Environment promotion becomes a Git operation.** Promoting a build from staging to
   production becomes "merge/copy this change in the GitOps repo," which is auditable,
   revertible, and doesn't require re-running CI or rebuilding an image — the same image
   is promoted across environments, which is a core GitOps and supply-chain integrity
   principle (build once, deploy the same artifact everywhere).
4. **Matches how Argo CD is designed to be used.** Argo CD Applications point at a
   Git source of manifests; pointing many Argo CD Applications at scattered
   locations across every app repo is workable but loses the single "here is
   everything currently deployed" view that a dedicated GitOps repo provides.

## Alternatives Rejected
- **Single-repo (manifests alongside app code)**: simpler to start with, and valid for
  very small setups, but breaks down as soon as you want independent promotion between
  environments or want to avoid the app repo's CI reacting to its own deployment
  commits.

## Consequences
- Requires an extra repository and a defined mechanism for CI to write to it (a
  "commit-back" step or an image-updater tool) — this is a real integration point we
  will build and document explicitly in Phase 7, not hand-wave.
- Requires developers (and us) to think in terms of "code change vs. deployment event"
  as two distinct actions, which is a deliberate part of what this project is meant to
  teach.
