# Changelog

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
