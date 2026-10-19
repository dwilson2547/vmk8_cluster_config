# Redis Cluster experiment notes (portable_cluster slice 4)

**Date:** 2026-10-19, vmk8 cluster, bitnami redis-cluster chart 13.0.4
(Redis 8.2.1, `bitnamilegacy` image), 6 nodes = 3 masters + 3 replicas on
2Gi local-path PVs, password via hand-applied Secret.

## Results (all verified)

- **Cluster formed**: `[OK] All 16384 slots covered`, `cluster_state:ok`,
  6 known nodes, 3 shards (slot ranges 0-5460, 5461-10922, 10923-16383),
  one replica per master.
- **Data spread**: 100 keys SET via cluster client; per-node DBSIZE shows
  the ~1/3 shard split.
- **Master-kill test**: deleted pod-0 (master of slots 0-5460); its replica
  was promoted; all sampled keys readable through the event, `cluster_state`
  back to `ok` within seconds; the restarted old master rejoined and
  reclaimed its range (standard failback) with the replica restored.

## Gotchas hit (for future sessions)

- **Bitnami free images are archived**: `docker.io/bitnami/redis-cluster:*`
  404s (Aug 2025 cutover). Use `docker.io/bitnamilegacy/<image>:<same tag>`
  — frozen, no updates, fine for the disposable lab. Repoint via
  `image.repository`.
- **Initial cluster creation comes from pod 0**, not a job: the chart's
  update Job only renders with `cluster.update.addNodes: true` (false by
  default). The sts pod-0 args set `REDIS_CLUSTER_CREATOR=yes` and its
  entrypoint runs `--cluster create` after all nodes are reachable. A
  wedged bootstrap shows as Readiness `cluster_state:fail` on every pod.
- **Stale `nodes.conf` wedges re-bootstrap**: if nodes ever ran with a
  partial/failed cluster config, the persisted `nodes.conf` makes the creator
  skip and pods sit at `cluster_state:fail` forever. Reset procedure:
  scale the sts to 0, delete the PVCs (`redis-data-<release>-<sts>-N` —
  note the name prefix), scale back up. Argo selfHeal rescales the sts
  almost immediately after a manual `kubectl scale`, so delete the PVCs
  during the scaled-down window.
- **The update Job is a helm hook by default** (`updateJob.helmHook:
  post-upgrade`): under ArgoCD it would be deferred to PostSync, past any
  readiness gate that depends on the cluster existing. Set it to `""` so
  Argo applies it as a regular resource — required for future addNodes
  scale-ups via GitOps.
- The chart's password default is helm `lookup` + random — pinned to the
  hand-applied `redis-cluster-password` Secret from the start
  (the Garage rpcSecret lesson, applied preemptively).
