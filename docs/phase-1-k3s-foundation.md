# Phase 1 — Local Kubernetes Foundation (k3s)

## Goal
Stand up a single-node k3s cluster on a dedicated Ubuntu Server VM, sized per
[ADR-005](adr/005-hardware-constraint-8gb-ram.md), and verify it's genuinely healthy —
not just that the install script exited without error.

## VM Configuration
Created via `VBoxManage` CLI (scripted, not GUI wizard) for reproducibility.

| Setting | Value |
|---|---|
| VM name | `idp-k3s-node` |
| RAM | 3072 MB |
| CPUs | 2 |
| Disk | 40 GB (dynamically allocated VDI; LVM currently provisions ~19GB of it — expandable later via `lvextend`, not yet needed) |
| OS | Ubuntu Server 26.04.1 |
| NIC 1 | NAT (internet access for package/image pulls) |
| NIC 2 | Host-only, `VirtualBox Host-Only Ethernet Adapter #2`, static IP `192.168.68.10/24` |

## Networking
- NIC1 (`enp0s3`): DHCP via NAT, used for outbound internet only
- NIC2 (`enp0s8`): static IP `192.168.68.10` via netplan, used for SSH access from the
  Windows host — isolated host-only subnet, no gateway configured (not routable, by design)

## k3s Install
```bash
curl -sfL https://get.k3s.io | sh -
```
Installed as a systemd service (`k3s.service`), single-node cluster acting as both
control-plane and worker (per ADR-001). Default bundled components (Traefik ingress,
local-path storage provisioner, CoreDNS) left enabled — not yet evaluated for
removal; will revisit only if RAM pressure becomes real under actual workloads.

## Verification
```bash
sudo systemctl status k3s     # active (running)
sudo k3s kubectl get nodes
```
```
NAME           STATUS   ROLES           AGE   VERSION
idp-k3s-node   Ready    control-plane   37s   v1.36.4+k3s1
```
`STATUS: Ready` confirms kubelet is healthy and the node registered successfully.

## Resource Evidence (per ADR-005)

| Metric | Before k3s | After k3s | Delta |
|---|---|---|---|
| RAM used | 387 Mi | 794 Mi | **+~407 Mi** |
| RAM available | 2.2 Gi | 1.8 Gi | -400 Mi |
| Disk used | 5.9 G | 6.4 G | **+~500 MB** |

Matches ADR-005's predicted 500MB–1GB steady-state control-plane cost. Comfortable
headroom remains within the 3GB VM allocation for Phase 2's application workload.

## Known Deviation
LVM logical volume is 19GB, not the full 40GB physical disk allocated to the VM —
default Ubuntu guided-LVM install behavior. Non-blocking; 12GB free is sufficient for
this project's scope. Documented here rather than silently ignored.

## Portfolio Evidence
- `k3s kubectl get nodes` output (above)
- Resource before/after table (above) — demonstrates deliberate capacity planning,
  not guesswork
