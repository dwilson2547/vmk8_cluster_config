---
title: helm lookup secrets are not render-stable under ArgoCD
date: 2026-10-18
tags: helm,argocd,gitops,secrets
---

Charts that generate secrets via helm lookup + randHex fallback will silently regenerate the value on any render where the lookup returns empty (ArgoCD repo-server renders without full cluster access), partitioning every workload initialized with the old value — seen with the garage chart's rpcSecret: restarted pods held the new secret, original pods the old one, RPC handshakes failed both ways. Rule: reference pre-existing secrets by name (existingRpcSecret etc.) for anything identity- or security-bearing under GitOps. Diagnosis trick: restart a never-restarted peer; if fresh pods interconnect and only disagree with old ones, a shared credential changed between init generations. See: docs/garage-experiment-notes.md
