---
tier: system
domains: [infra]
---

# vmk8_cluster_config

Kubernetes manifests and ArgoCD application definitions for the **vmk8 dev cluster** — the
MicroK8s cluster running on the Epyc HPC box as 5 VMs. Dev workloads live here; the physical
cluster (`infra/cluster-config`, pool 192.168.0.60-85) stays prod.

## Cluster

| Node | IP (assigned at the router) | Role |
|---|---|---|
| vmk8main | 192.168.0.90 | control plane |
| vmk8node1 | 192.168.0.91 | worker |
| vmk8node2 | 192.168.0.92 | worker |
| vmk8node3 | 192.168.0.93 | worker |
| vmk8node4 | 192.168.0.94 | worker |

- MicroK8s `1.33/stable` (snap), containerd, Calico CNI.
- Kubeconfig lives at `~/vmk8cluster/config` on the workstation (never commit it).
- SSH aliases: `vmk8main`, `vmk8node1-4`.

## Network allocation (whole LAN)

| Range / IP | Owner |
|---|---|
| 192.168.0.60-85 | prod MetalLB pool (`infra/cluster-config/metallb/pool.yaml`) |
| 192.168.0.61 | prod CoreDNS (zone authority for `*.local`) |
| 192.168.0.65 | prod public Traefik (WAN forward) |
| 192.168.0.90-94 | vmk8 nodes (router-assigned) |
| 192.168.0.95-99 | vmk8 MetalLB pool (this repo, `metallb/pool.yaml`) |

Never let the two pools overlap: both announce via L2/ARP on the same LAN.

## DNS

The LAN is migrating off the RFC 6762-reserved `.local` TLD to `home.arpa`
(RFC 8375): prod CoreDNS at 192.168.0.61 serves a consolidated `home.arpa`
zone, with this cluster's ingress at `*.vmk8.home.arpa` → 192.168.0.95. The
`.local` zones are dual-served until all references are migrated.

Resolution chain for this cluster:

- Hosts: systemd-resolved → 192.168.0.61 first (zone authority + upstream
  forward to 8.8.8.8/1.1.1.1). `home.arpa` names resolve natively — no
  systemd-resolved special-casing, unlike `.local`.
- Pods: in-cluster CoreDNS is pinned with `forward . 192.168.0.61`
  (hand-applied patch, see below). It must NOT use `forward . /etc/resolv.conf`:
  the node's resolv.conf lists the router (192.168.0.1) too, and CoreDNS's
  default random upstream policy would send ~half of pod queries to the router,
  which NXDOMAINs every `home.arpa` name.

The coredns patch is a documented manual step (the kube-system CoreDNS belongs
to the microk8s dns addon; it is not ArgoCD-managed):

```bash
kubectl -n kube-system edit cm coredns   # forward . /etc/resolv.conf → forward . 192.168.0.61
kubectl -n kube-system rollout restart deploy/coredns
```

## Bootstrap

ArgoCD itself is installed by hand once (upstream install manifest, namespace `argocd`), then:

```bash
kubectl apply -f bootstrap/apps.yaml   # with KUBECONFIG=~/vmk8cluster/config
```

After that, everything deploys by git push to `main` — same workflow as the prod repo.
