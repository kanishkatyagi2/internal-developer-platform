# ADR-006: GitHub Container Registry (GHCR) as the image registry


  ## Status
  Accepted (2026-10-04)

## Related
- ADR-002 — Two-repo GitOps
- ADR-005 — 8GB RAM ceiling

## Context

Until Phase 5, the path from code to cluster was manual: `docker save`,
`scp`, `k3s ctr images import`. CI produced only a temporary workflow
artifact that nothing in Kubernetes could pull. The image CI tested was
not the image the cluster ran. Phase 7 (Argo CD) needs images that can
be pulled by name and traced to a commit.

## Decision

1. **Registry: GHCR (`ghcr.io`).** It is free at this scale, lives next
   to the code, and needs no additional service on the 8GB machine.
2. **Package visibility: private.** Application images are private in
   most real organizations. The first package published by the workflow inherited the public visibility of the source repository and was subsequently switched to private.
3. **Tags: full commit SHA only. No `latest`.** Tags are traceable to a
   commit and not reused for different content. This gives meaningful
   GitOps diffs and rollbacks.
4. **CI authentication: `GITHUB_TOKEN`** with job-level
   `packages: write`, and login and push only on pushes to `main`.
   No long-lived push credential exists.
5. **Cluster authentication: `imagePullSecret`** (`ghcr-pull-secret`)
   built from a classic personal access token with only `read:packages`,
   30-90 day expiry. GHCR does not support fine-grained tokens. The
   Secret is created manually with `kubectl` and never committed.
6. **Quality gate:** CI starts the built image and polls `/healthz`
   before login and push. A broken image fails the build and never
   reaches the registry.

## Alternatives Rejected

- **Docker Hub:** needs a separate account and a stored long-lived
  push token, plus anonymous pull rate limits.
- **Self-hosted (Harbor, registry:2):** common in industry but costs
  RAM and disk we do not have (ADR-005).
- **Public package:** simpler, but not how application images are
  handled in practice.

## Consequences

- No registry to run, so the resource impact on the cluster is effectively zero (only the pulled image layers on disk).
- The pull token is a personal credential with an expiry. It must be
  rotated by hand and is tied to one GitHub account. A machine
  account or GitHub App would replace it in a real organization.
  Revisit in Phase 11 (Secrets).
- Tags are immutable by convention, not enforcement. Pinning by image
  digest is a possible hardening step.
- The smoke test would have caught the Phase 6 incident where the
  image omitted `app.js` and crash-looped in the cluster, while every
  other CI step was green.