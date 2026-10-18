---
title: vmk8 dev cluster rebuilt and under GitOps
date: 2026-10-17
tags: k8s,argocd,infra
---

The vmk8 MicroK8s cluster (5 VMs on the Epyc box, 192.168.0.90-94, router-assigned) was rebuilt after a DHCP IP change crash-looped k8s-dqlite (root cause: docs/issues/2026_10_18_dhcp_ip_change_broke_dqlite.md). New public repo infra/vmk8-cluster-config (github: dwilson2547/vmk8_cluster_config) mirrors the prod cluster-config pattern: app-of-apps bootstrap, MetalLB pool 192.168.0.95-99 (disjoint from prod .60-.85), Traefik LAN-only ingress pinned to .95, ArgoCD installed upstream + bootstrap applied 2026-10-18. Host and pod DNS resolves .local zones via prod CoreDNS 192.168.0.61 once VM netplan resolvers list it first. Kubeconfig: ~/vmk8cluster/config.
