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

The vmk8 hosts resolve through prod CoreDNS at 192.168.0.61 first (it serves every `.local`
zone and forwards unknown names to 8.8.8.8/1.1.1.1). MicroK8s in-cluster CoreDNS forwards to
`/etc/resolv.conf`, so pods inherit that chain automatically — no in-cluster DNS config needed.
Dev zones that should point at vmk8 ingress get their A records repointed to 192.168.0.95 in
`infra/cluster-config/dns/dns.yaml` (prod repo stays the single DNS authority).

## Bootstrap

ArgoCD itself is installed by hand once (upstream install manifest, namespace `argocd`), then:

```bash
kubectl apply -f bootstrap/apps.yaml   # with KUBECONFIG=~/vmk8cluster/config
```

After that, everything deploys by git push to `main` — same workflow as the prod repo.
