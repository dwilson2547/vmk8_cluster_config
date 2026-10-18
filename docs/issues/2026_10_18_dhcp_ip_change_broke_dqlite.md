# DHCP lease change moved the control-plane IP, crash-looping k8s-dqlite and blocking the API server

**Date:** 2026-10-18  
**Component:** `/var/snap/microk8s/current/var/kubernetes/backend/` — dqlite cluster address; also affected kube-apiserver certs and the copied kubeconfig  
**Severity:** High — entire cluster unavailable; datastore could not start (93 restarts)

---

## Observed symptom

After a routine OS update and reboot of all five VMs, `kubectl get nodes` on vmk8main returned
`connection refused` on `127.0.0.1:16443`. `microk8s status` reported not running, and
`ss -tln` showed nothing bound to 16443.

---

## Root cause

### The VM's IP changed across the reboot

The VMs received DHCP leases; vmk8main held 192.168.0.70 when the cluster was created but came
back from the reboot as 192.168.0.90. Nothing else was wrong — all services were healthy before
the reboot.

### dqlite pins the address it was created with

The MicroK8s datastore (k8s-dqlite) binds the dqlite cluster port on the node IP recorded at
cluster creation:

```
level=fatal msg="Failed to create server"
error="failed to create dqlite app: listen to 192.168.0.70:19001:
listen tcp 192.168.0.70:19001: bind: cannot assign requested address"
```

With the old IP gone, dqlite crash-looped (restart counter reached 93), so `kine.sock` never
listened and kube-apiserver — which waits on the datastore — never bound 16443. The same
recorded .70 address explained the copied kubeconfig (server `https://192.168.0.70:16443`)
failing from the workstation.

---

## Troubleshooting steps taken

1. **SSH and uptime on all five VMs** — ruled out VM-level failure; all up post-reboot, only the cluster was down.
2. **`snap services microk8s`** — showed kubelite/containerd/cluster-agent active but etcd and apiserver-proxy inactive; confirmed partial start, not a clean stop state.
3. **`ss -tln` for 6443/16443** — nothing bound; ruled out slow startup alone.
4. **`journalctl -u snap.microk8s.daemon-kubelite`** — kube-apiserver repeatedly failing to dial `kine.sock` (connection refused); pointed at the datastore, not the API server itself.
5. **`journalctl -u snap.microk8s.daemon-k8s-dqlite`** (daniel is in `adm`, so readable without sudo) — found the fatal bind error on 192.168.0.70:19001 and the restart counter at 93; confirmed root cause.
6. **Checked 192.168.0.70 reachability and workspace MetalLB pool** — .70 was free, but `infra/cluster-config/metallb/pool.yaml` reserves .60-.85 for the prod cluster, so simply restoring .70 was not an option.

---

## Fix

### Cluster — wiped and rebuilt on stable addresses

The cluster held nothing of value, so rather than regenerate dqlite cluster state, API server
certs and kubeconfigs for a new IP, MicroK8s was removed on all five nodes
(`sudo snap remove --purge microk8s`) and rebuilt on 192.168.0.90-.94 — addresses assigned
manually at the router, outside the prod MetalLB pool and stable across reboots. A fresh
kubeconfig was generated from the rebuilt cluster.

### Prevention — stable control-plane addressing

Cluster-creating components (dqlite, kube-apiserver certs, join configs) record the node IP at
creation; an address that can change on reboot makes the cluster unbootable. Router-assigned
static addresses (or netplan statics) are a precondition for a stable cluster, and this repo now
documents the LAN allocation table so prod pool, vmk8 nodes and the vmk8 MetalLB pool (.95-.99)
never overlap.

---

## Files changed

- `README.md` — network allocation table documenting the disjoint address ranges
- `CLAUDE.md` — rule: MetalLB pool must stay disjoint from prod and node addresses
