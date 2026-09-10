# Phase 3 — Kubernetes Deployment (Manual Manifests)

## Goal
Deploy `hello-service` onto the k3s cluster using raw Kubernetes manifests (Deployment +
Service), with no Helm abstraction yet, and verify the full request path from outside
the cluster to a running pod.

## Problem Solved First: Local Image Availability
No container registry exists yet (Phase 6). The image built in Phase 2 lived only in
Docker Desktop's store on Windows — a separate machine from the k3s VM's containerd.

Resolved via manual transfer:
```bash
# Windows
docker save hello-service:0.1.0 -o hello-service.tar
scp hello-service.tar platform@192.168.68.10:/home/platform/

# VM
sudo k3s ctr images import hello-service.tar
```
Verified with `sudo k3s ctr images ls | grep hello-service`.

This is intentionally manual and temporary — the exact problem container registries
solve. Replaced by GHCR in Phase 6.

## Deployment Manifest
`kubernetes/manifests/hello-service-deployment.yaml` — key decisions:
- `replicas: 1` (ADR-005 — no unnecessary replication)
- `imagePullPolicy: Never` (image only exists via local import, no registry to pull from)
- Small explicit `resources.requests/limits` (32Mi/50m request, 64Mi/100m limit)
- `livenessProbe` + `readinessProbe` both hitting `/healthz` (built in Phase 2
  specifically for this)

## Service Manifest
`kubernetes/manifests/hello-service-service.yaml` — `NodePort` type, exposing the app
on the node's own IP at a fixed port (`30080`), appropriate for a local lab without
Ingress/DNS setup yet.

## Verification

```
$ sudo k3s kubectl get pods,svc,deploy -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP           NODE
pod/hello-service-58f8667bb9-542z6   1/1     Running   0          15m   10.42.0.14   idp-k3s-node

NAME                    TYPE       CLUSTER-IP     PORT(S)          SELECTOR
service/hello-service   NodePort   10.43.201.35   3000:30080/TCP   app=hello-service

NAME                            READY   UP-TO-DATE   AVAILABLE   IMAGES
deployment.apps/hello-service   1/1     1            1           docker.io/library/hello-service:0.1.0
```

End-to-end request test, from Windows through the host-only network to the pod:
```
curl.exe http://192.168.68.10:30080/
{"service":"hello-service","message":"Hello from the golden path","version":"0.1.0"}

curl.exe http://192.168.68.10:30080/healthz
{"status":"ok"}
```

## Resource Evidence (per ADR-005)

| Metric | Phase 1 baseline (k3s only) | After hello-service deployed | Delta |
|---|---|---|---|
| RAM used | 794 Mi | 1.0 Gi | +~230 Mi |
| RAM available | 1.8 Gi | 1.5 Gi | -300 Mi |
| Swap used | 0 B | 35 Mi | +35 Mi (minor, noted not concerning) |

Well within the 3GB VM budget with room for Phase 4/5 work.

## What This Phase Taught (concepts, not just commands)
- **Deployment → Pod linkage is label-based**, not name-based — this indirection is
  what makes rolling updates possible later
- **Service → Pod linkage is a separate label-based match**, independent of the
  Deployment's own selector, even though both point at the same labels in practice
- **Readiness vs. liveness probes are functionally different**: liveness triggers
  container restarts, readiness controls whether the Service sends traffic at all —
  distinction becomes concretely important in Phase 13 (rollout/rollback)

## Portfolio Evidence
- `get pods,svc,deploy -o wide` output (above) — shows correctly linked objects
- Both curl responses proving external reachability
- Resource before/after table (above)
