---
title: Garage slice of portable_cluster deployed and verified
date: 2026-10-18
tags: s3,garage,portable-cluster
---

Garage v2.4.1 on vmk8 (ArgoCD app, upstream chart from deuxfleurs-org/garage): 3 replicas on local-path PVCs, k8s-native discovery, 256 partitions 3x replicated. Verified: S3 write/read, object readable during a node kill, pod recreated on the same node with PVC intact. Gotchas recorded in docs/garage-experiment-notes.md: bucket allow --owner does not grant read/write; from-scratch image (binary at /garage); aws-cli kubectl run needs --command. See: docs/garage-experiment-notes.md
