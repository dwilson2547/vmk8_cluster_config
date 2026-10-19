---
title: CockroachDB helm chart init Job deadlocks under ArgoCD
date: 2026-10-19
tags: cockroachdb,helm,argocd
---

The official cockroachdb helm chart's init Job carries helm.sh/hook: post-install; ArgoCD runs it as a PostSync hook only after all resources are healthy — but the CRDB StatefulSet cannot become healthy until the init Job (cockroach init) runs. Fresh GitOps deploys with this chart hang at pods 'waiting for init'. Break it manually: kubectl exec cockroachdb-0 -- /cockroach/cockroach init --insecure --host=<pod dns>; pods go Ready, sync completes, and the hook job then Completes without wedging the app. Verified: range replicas on {1,2,3}, INSERTs held through a node kill, zero loss. See: docs/cockroachdb-experiment-notes.md
