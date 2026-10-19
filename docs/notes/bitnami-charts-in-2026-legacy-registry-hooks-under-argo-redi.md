---
title: bitnami charts in 2026: legacy registry, hooks under Argo, redis-cluster bootstrap
date: 2026-10-19
tags: redis,bitnami,argocd
---

Bitnami free images were archived Aug 2025: docker.io/bitnami/* 404s; use docker.io/bitnamilegacy/<image>:<same tag> (frozen). The redis-cluster chart: initial cluster creation is done by pod-0 (REDIS_CLUSTER_CREATOR in sts args), not a job; a stale nodes.conf wedges re-bootstrap (pods stuck cluster_state:fail) — reset by scaling the sts to 0 and deleting PVCs (named redis-data-*) in the window. The updateJob is a helm hook (deferred to PostSync under Argo) — set updateJob.helmHook: '' to make it a regular resource for scale-ups. Password: pin existingSecret, bitnami generates via helm lookup otherwise. Verified: 3 shards + replicas, master kill → promotion, zero loss. See: docs/redis-experiment-notes.md
