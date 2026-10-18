# Changelog

## 2026-10-18 - docs: NFS teardown-wedge issue record

### Added

- `docs/issues/2026_10_18_nfs_teardown_stuck_after_sigkill.md` — SIGKILL'd pod
  left the kubelet CSI mount dir populated, wedging pod/PVC/PV deletion
  (`directory not empty` retry loop). Remediation: empty the mount dir as root,
  kubelet's retry completes the teardown. Applies to prod too (same driver
  and StorageClass).

## 2026-10-18 - storage: NFS CSI driver + nfs-dataset StorageClass

### Added

- `argocd/nfs-csi.yaml` — upstream csi-driver-nfs chart pinned to v4.13.2,
  matching the prod cluster's driver version.
- `argocd/nfs-dataset.yaml` + `storage/nfs-dataset.yaml` — `nfs-dataset`
  StorageClass mirroring prod exactly: TrueNAS 192.168.0.10,
  `/mnt/ssd_pool/nfs_dataset`, hard NFSv4.1, Delete reclaim. Both clusters
  provision into the same share as distinct subdirectories.

## 2026-10-18 - dns: migrate dashboard host to home.arpa; pin pod DNS upstream

### Changed

- `traefik/values.yaml`: dashboard host `traefik.vmk8.local` → `traefik.vmk8.home.arpa`
  (`.local` is RFC 6762-reserved for mDNS and unusable from systemd-resolved hosts;
  prod CoreDNS now serves a consolidated `home.arpa` zone with `*.vmk8` → 192.168.0.95).

### Fixed

- Pod DNS for `home.arpa` names was nondeterministic (NXDOMAIN ~half the time):
  the node resolv.conf lists the router as a secondary resolver, and in-cluster
  CoreDNS's default random forward policy sent roughly half of pod queries to the
  router, which does not know the zone. Pinned `forward . 192.168.0.61` in the
  kube-system coredns ConfigMap (documented manual patch, see README — the
  configmap belongs to the microk8s dns addon, so it is not ArgoCD-managed).
  Verified 3/3 pod lookups after the rollout.

## 2026-10-18 - traefik: unblock first sync

### Fixed

- The traefik Application's first automated sync failed with
  `namespaces "ingress" not found` — the namespace does not pre-exist on a fresh
  cluster (it does on prod, which is why the mirrored manifest had no
  CreateNamespace). Added `CreateNamespace=true` to `argocd/traefik.yaml` and
  created the namespace by hand to clear the in-flight retry.

## 2026-10-18 - initial scaffold: app-of-apps, MetalLB, Traefik

### Added

- `bootstrap/apps.yaml` — app-of-apps root (the only manifest applied by hand).
- `argocd/metallb.yaml`, `argocd/traefik.yaml` — first two Applications.
- `metallb/` — upstream v0.15.2 manifest + kustomization (same pattern as prod) with the
  vmk8 dev pool `192.168.0.95-99`, disjoint from the prod pool (.60-.85) and the vmk8 node
  addresses (.90-.94).
- `traefik/values.yaml` — LAN-only dev ingress, chart pinned to 37.4.0 (parity with prod),
  LoadBalancer pinned to 192.168.0.95.
- `README.md`, `CLAUDE.md` — topology, LAN allocation table, DNS chain
  (hosts/pods resolve via prod CoreDNS at 192.168.0.61), dev/prod split rules.
- `docs/issues/2026_10_18_dhcp_ip_change_broke_dqlite.md` — root cause record for the
  IP-change incident that took down the pre-rebuild cluster and motivated the rebuild
  with router-assigned static addresses.
