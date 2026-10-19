# CockroachDB experiment notes (portable_cluster slice 3)

**Date:** 2026-10-19, vmk8 cluster, CockroachDB 25.4.4 via the official helm
chart 19.0.6 (pinned to upstream tag `v25.4.4-preview+1`), 3 replicas on
10Gi local-path PVs, insecure mode (dev cluster).

## Results (all verified)

- **Cluster Ready**: 3 nodes on 3 different nodes, gossip shows node_ids
  {1,2,3}.
- **Data + range spread**: `portable.cluster_test` (100 rows) — its range
  replicated on nodes {1,2,3} (`SHOW RANGES ... WITH DETAILS`).
- **Node-kill test**: deleted `cockroachdb-1`'s pod; **INSERTed 20 rows
  during the outage** (raft majority 2/3 held), reads continued, pod
  rejoined Ready with its PVC, final count 120/120 with all 20 outage rows
  durable, range replicas fully re-spread {1,2,3}.

## Gotchas hit (for future sessions)

- **The chart's init Job is a `helm.sh/hook: post-install` — under ArgoCD
  this deadlocks**: Argo runs it as a PostSync hook only after all resources
  are healthy, but the StatefulSet can't become healthy until the job
  (`cockroach init`) runs. Pods sat "waiting for init" forever. Workaround:
  run the init manually, exactly as the job would:
  ```bash
  kubectl -n cockroachdb exec cockroachdb-0 -- /cockroach/cockroach init \
    --insecure --host=cockroachdb-0.cockroachdb.cockroachdb.svc.cluster.local
  ```
  After manual init the pods go Ready and Argo completes the sync; the
  PostSync hook job then runs and Completes (does not wedge the app).
  Expect this on any fresh CRDB deploy via GitOps with this chart.
- **First boot is slow on the ZFS/SATA local tier** — store-format upgrade
  steps each pay the sync-write tax (minutes before nodes serve). The
  benchmark's caveat applies: fine for topology testing, not representative
  of NVMe nodes.
- `SHOW RANGES FROM TABLE` needs `WITH DETAILS` for the replicas/lease
  columns in 25.4.
- Insecure mode (`tls.enabled: false`) is a deliberate dev-lab tradeoff;
  the chart's default TLS self-signer is the real-deployment path.
