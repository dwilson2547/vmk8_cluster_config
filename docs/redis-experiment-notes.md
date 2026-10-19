# Redis Cluster experiment notes (portable_cluster slice 4)

**Date:** 2026-10-19, vmk8 cluster. **Revision 2 (same day):** the initial
bitnami-chart deploy (frozen `bitnamilegacy` image) was replaced with the
OT-CONTAINER-KIT redis-operator v0.27.0 + `quay.io/opstree/redis:v8.10.1`
(active project, actively rebuilt image, Redis 8.10 line) — developing on
archived images is a landmine in a blueprint, even when the security surface
of a LAN lab is forgiving. Both runs' results and gotchas are kept: the
failure-mode lessons carry over, the bitnami config is history.

## Results (operator run — all verified)

- **Deploy**: `RedisCluster/lab` CR via the project's own
  `charts/redis-cluster` (operator at `charts/redis-operator`, both pinned
  to v0.27.0); 3 leaders + 3 followers across 4 nodes; per-pod data (2Gi)
  + node-conf (1Gi) PVCs on local-path; password from the same hand-applied
  `redis-cluster-password` Secret.
- **Cluster**: `cluster_state:ok`, 16384 slots across 3 shards (0-5460,
  5461-10922, 10923-16383), 6 cluster nodes, one replica per master.
- **Master-kill test**: deleted `lab-leader-0` (master of 0-5460); its
  follower was **promoted to master of the range**; the restarted old
  leader rejoined and demoted itself to slave of its former replica; all
  sampled keys readable, `cluster_state:ok` throughout, zero loss.
- **Transition hygiene**: swapping the chart source re-rendered the app to a
  single RedisCluster CR; Argo pruned the old bitnami StatefulSet
  automatically; only the sts-created PVCs needed a manual delete (they are
  not Argo-tracked).

## Gotchas hit (bitnami run — historical, kept for the lessons)

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
