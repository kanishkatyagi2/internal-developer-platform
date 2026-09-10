# Phase 2 — Containerized Application (`hello-service`)

## Goal
Build the first golden-path reference application: a minimal Express app, containerized
via a multi-stage Dockerfile, proven to build and run correctly — before any Kubernetes
involvement.

## Repository
Separate repo per [ADR-002](adr/002-two-repo-gitops-model.md): `hello-service`
(https://github.com/<your-username>/hello-service), not inside this platform repo.

## Application
Two endpoints:
- `GET /` — returns service metadata as JSON
- `GET /healthz` — returns `{"status":"ok"}`, used later by Kubernetes liveness/readiness probes (Phase 3)

## Dockerfile Design Decisions
- **Multi-stage build**: separates dependency install from the final runtime image,
  keeping the shipped image smaller (excludes npm cache/build tooling)
- **Non-root user** (`appuser`): security practice established from the first service,
  rather than retrofitted in Phase 15
- **`node:20-alpine` base**: small footprint, appropriate for the project's resource
  constraints (ADR-005)
- **Explicit version tag** (`0.1.0`, not `latest`): avoids the "what's actually
  deployed?" ambiguity that `latest` tags cause in real incidents

## Verification
```
docker build -t hello-service:0.1.0 .
docker run -p 3000:3000 hello-service:0.1.0
```
```json
{"service":"hello-service","message":"Hello from the golden path","version":"0.1.0"}
{"status":"ok"}
```

## Image Size
```
IMAGE                 DISK USAGE   CONTENT SIZE
hello-service:0.1.0   199MB        49.1MB
```

## Known Issue Encountered
Initial `docker build` failed with a TLS handshake timeout pulling `node:20-alpine`
from Docker Hub — transient network issue, resolved on retry. Not a Dockerfile or
project configuration problem; noted here in case it recurs.

## Open Problem Carried Into Phase 3
This image exists only in Docker Desktop's local store on Windows. The k3s VM has its
own separate containerd with no access to it. No registry exists yet (that's Phase 6),
so Phase 3 will use manual `docker save` / `k3s ctr images import` to get this specific
image onto the node — a deliberate simplification, replaced by GHCR once Phase 6 is
reached.

## Portfolio Evidence
- Build output, run output, both endpoint responses (above)
- `docker images hello-service` output (above)
