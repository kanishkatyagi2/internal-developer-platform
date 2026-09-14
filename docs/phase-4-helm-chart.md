# Phase 4 — Helm: Parameterizing hello-service

## Goal

Convert the hand-written Kubernetes manifests from Phase 3 (`hello-service-deployment.yaml`, `hello-service-service.yaml`) into a single, reusable, parameterized Helm chart — the first step toward a real "golden path" for services on this platform, rather than one-off YAML per app.

## Real-world problem being solved

Static manifests don't scale past one service in one environment:

- No native templating or reuse mechanism in raw Kubernetes YAML — every new service or environment means copy-pasting and hand-editing full manifests.
- Multiple environments (dev/staging/prod) need different replica counts, resource limits, and image tags without duplicating the underlying object structure.
- A platform intended to onboard other services (Phase 10 — Golden Path templates) needs one blessed, reusable deployment pattern, not a manifest per app.

Helm addresses this by splitting a manifest into a **template** (structure) and **values** (the specifics), the same problem Terraform/Ansible solve for infrastructure. This is also the standard way third-party software (Prometheus, cert-manager, Argo CD) is installed on Kubernetes, and it's what Argo CD (Phase 7) will point at per environment.

## Architecture

```
helm/service-chart/
├── Chart.yaml           # chart metadata
├── values.yaml           # default configuration values
├── .helmignore
└── templates/
    ├── deployment.yaml    # templated Deployment
    ├── service.yaml        # templated Service
    └── _helpers.tpl         # reusable naming/label templates
```

`service-chart` is deliberately generic — `hello-service` is a *value* (`nameOverride`), not hardcoded into any template — so future services can reuse this exact chart with their own `values.yaml`.

## Why each design decision

- **One generic chart, not one chart per service** — a chart per service would reproduce the copy-paste problem Helm exists to solve. A single parameterized chart is the actual golden-path pattern.
- **`_helpers.tpl` for naming/labels** — Service `selector` and Pod `labels` must match exactly for service discovery to work; centralizing that logic in one helper avoids drift.
- **`selectorLabels` kept as a subset of `labels`** — Service selectors should stay minimal and stable so a future label addition (e.g. a version label) can't silently break Pod selection.
- **Resource requests/limits stay explicit in `values.yaml`** — per ADR-005, nothing is resource-unbounded; defaults are small (`50m/100m` CPU, `64Mi/128Mi` memory) but held as overridable values, not baked into templates.
- **`imagePullPolicy: Never` retained** — Phase 6 (GHCR) hasn't happened yet, so the chart still targets the manually-imported image from Phase 3.

## Implementation

`Chart.yaml`, `values.yaml`, `_helpers.tpl`, `templates/deployment.yaml`, `templates/service.yaml` written to define a parameterized Deployment + NodePort Service, using `include`/`nindent` for labels and `.Values` for every environment-specific field (image repo/tag, replica count, service port/nodePort, resource requests/limits, liveness/readiness probe paths and timing).

## Troubleshooting journey (real issues hit and resolved)

This phase surfaced several genuine environment/config issues worth documenting as portfolio evidence of real debugging, not just a scripted happy path:

1. **`fullname` template collapsed to the chart name** — initial `_helpers.tpl` didn't correctly incorporate `.Release.Name`, causing rendered object names to be `service-chart` instead of `hello-service`. Fixed the `fullname` template logic and set `nameOverride: "hello-service"` in `values.yaml`.
2. **Directory structure confusion** — `helm create service-chart` was run from the wrong working directory, producing a nested `helm/service-chart/service-chart/` folder. Diagnosed via `Get-ChildItem -Recurse` and flattened by moving the inner folder's contents up one level.
3. **Repo never cloned onto the VM** — all prior work assumed a git checkout existed on `idp-k3s-node`; it didn't. Cloned `internal-developer-platform` onto the VM, then `scp`'d the not-yet-pushed `helm/service-chart/` folder on top of it.
4. **`scp` destination didn't exist** — first `scp` attempt failed because `~/internal-developer-platform` didn't exist yet on the VM (see #3); resolved by cloning first.
5. **Helm not installed on the VM** — installed via the official `get-helm-3` script (not `snap`, to avoid unnecessary confinement/container overhead on an 8GB host). Verified `v3.22.0`.
6. **`kubectl` permission denied without `sudo`** — `/etc/rancher/k3s/k3s.yaml` is `600 root:root`. Root cause was two-fold: (a) no user-owned kubeconfig existed yet, and (b) `kubectl` on this system is a **symlink to the `k3s` binary** (`/usr/local/bin/kubectl -> k3s`), which hardcodes `/etc/rancher/k3s/k3s.yaml` as its config path unless `KUBECONFIG` is explicitly exported — it does not fall back to `~/.kube/config` like the standalone `kubectl` binary would. Fixed by copying `k3s.yaml` to `~/.kube/config` with correct ownership/mode (`600`, user-owned) and exporting `KUBECONFIG=~/.kube/config` (made permanent via `~/.bashrc`).
7. **Image reference mismatch** — chart's default `image.repository: hello-service` didn't match the actual imported image tag `docker.io/library/hello-service:0.1.0` (confirmed via `sudo k3s ctr images ls`). Fixed by setting `values.yaml`'s `image.repository` to `docker.io/library/hello-service`, avoiding an otherwise-silent `ImagePullBackOff` at install time.

## Verification steps & expected output

```bash
helm lint .
# ==> Linting .
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed

helm template hello-service . --namespace default
# renders Deployment + Service, all names/labels = hello-service,
# image = docker.io/library/hello-service:0.1.0

# removed Phase 3 manifests first:
kubectl delete deployment hello-service
kubectl delete service hello-service

helm install hello-service . --namespace default
# STATUS: deployed, REVISION: 1

kubectl get pods,svc
# pod/hello-service-666dcf855b-bzs69   1/1   Running
# service/hello-service   NodePort   80:30080/TCP

curl -UseBasicParsing http://192.168.68.10:30080/healthz
# StatusCode: 200, Content: {"status":"ok"}

helm list
helm status hello-service
# STATUS: deployed, CHART: service-chart-0.1.0, APP VERSION: 0.1.0
```

## Resource evidence (ADR-005)

| | Memory available | Disk available |
|---|---|---|
| Before Helm install | 1.6Gi | 9.9G |
| After Helm install | 1.6Gi | 9.9G |

Negligible delta — Helm client itself adds no persistent overhead for a basic install (no cluster-side controller involved), and the re-deployed pod is the same single replica as Phase 3.

## How this maps to real industry practice

This is the same chart shape used by platform teams internally — a generic "web-service" chart every team's app installs against with its own `values.yaml`. Third-party software on Kubernetes (Prometheus, cert-manager, Argo CD itself) follows the identical pattern. Argo CD in Phase 7 will point at this same chart with per-environment values files.

## What was committed to git

```
helm/service-chart/Chart.yaml
helm/service-chart/values.yaml
helm/service-chart/.helmignore
helm/service-chart/templates/deployment.yaml
helm/service-chart/templates/service.yaml
helm/service-chart/templates/_helpers.tpl
docs/phase-4-helm-chart.md
```

`kubernetes/manifests/hello-service-*.yaml` from Phase 3 kept in the repo as a "before" reference, superseded by `helm/service-chart/` as of this phase.

## Portfolio evidence collected

- Clean `helm lint` output
- `helm template` render output
- `helm install` / `helm status` / `helm list` output
- `kubectl get pods,svc` post-install
- `curl /healthz` verification (200, `{"status":"ok"}`)
- Before/after `free -h` / `df -h`
- Documented root-cause debugging for kubeconfig permissions and image tag mismatch
