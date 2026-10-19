---
title: Strimzi 1.x on k8s: v1 CRDs, node pools, argocd recurse
date: 2026-10-18
tags: kafka,strimzi,argocd
---

Strimzi 1.x: Kafka and KafkaTopic CRDs serve v1 only (v1beta2 gone); broker topology must use KafkaNodePool CRs (spec.kafka.replicas removed) — combined mode is one pool with roles [controller, broker], storage on the pool. On ArgoCD, directory apps do not recurse subdirectories without directory.recurse: true — silently skips nested manifests (hit with kafka/topics/). Verified 3-node KRaft on local-path: broker kill during production held (min ISR 2), leaders failed over, 120/120 messages intact. See: docs/kafka-experiment-notes.md
