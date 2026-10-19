# Changelog

## 2026-10-19 - redis: replaced bitnami chart with redis-operator (maintained infra)

### Changed

- `redis/values.yaml` + `argocd/redis.yaml`: the bitnami redis-cluster chart
  (running the frozen `bitnamilegacy` image, archived Aug 2025) is replaced by
  the OT-CONTAINER-KIT redis-operator (v0.27.0, released 9 days ago) with
  `quay.io/opstree/redis:v8.10.1` — an actively rebuilt image on the current
  Redis line. Same shape (3 masters + 3 replicas), same hand-applied
  password secret, same local-path storage. Archived images in a blueprint
  are a landmine: no security patches ever, wrong default to copy.
- `argocd/redis-operator.yaml`: the operator app.

## 2026-10-19 - redis: experiment verified (slots, spread, master failover)

### Added

- `docs/redis-experiment-notes.md` — full run record: 16384 slots across
  3 shards + replicas, 100 keys sharded ~1/3 per master, master kill with
  replica promotion and zero data loss. Gotchas: bitnamilegacy registry,
  pod-0-as-cluster-creator, stale nodes.conf wedging re-bootstrap (reset
  via scale-0 + PVC delete), updateJob helm hook disabled for Argo.

## 2026-10-19 - redis: Redis Cluster (bitnami chart 13.0.4), 3+3 on local-path

### Added

- `argocd/redis.yaml` + `redis/values.yaml` — bitnami redis-cluster chart
  pinned to redis-cluster/13.0.4 (Redis 8.x): 6 nodes (3 masters + 3
  replicas), 2Gi local-path PVs, password pinned via hand-applied
  `redis-cluster-password` Secret (avoiding the bitnami lookup-generated
  password — the rpcSecret churn lesson applied preemptively).
  portable_cluster blueprint slice 4.

## 2026-10-19 - cockroachdb: experiment verified (range spread, writes under node kill)

### Added

- `docs/cockroachdb-experiment-notes.md` — full run record: table range
  replicated on {1,2,3}, 20 rows INSERTed during a node kill (raft majority
  held), 120/120 rows intact after rejoin. Gotcha: the chart's init Job is a
  helm post-install hook, which deadlocks under ArgoCD (PostSync runs only
  after healthy, cluster needs the job to become healthy) — manual
  `cockroach init` breaks the cycle; job then completes without wedging.

## 2026-10-19 - cockroachdb: CRDB 25.4.4, 3 replicas on local-path

### Added

- `argocd/cockroachdb.yaml` + `cockroachdb/values.yaml` — CockroachDB chart
  19.0.6 (CRDB 25.4.4) from the upstream git repo, 3-replica StatefulSet with
  10Gi local-path PVs, insecure mode (dev cluster; the chart's TLS self-signer
  is the real-deployment default). portable_cluster blueprint slice 3.

## 2026-10-19 - kafka: experiment verified (spread, round trip, broker kill)

### Added

- `docs/kafka-experiment-notes.md` — full run record: 6-partition topic
  spread across all three brokers, 100-message produce/consume, broker kill
  during production (min ISR 2 held, leaders failed over, 120/120 messages
  intact). Gotchas: Strimzi 1.x v1-only Kafka CRD, mandatory KafkaNodePool
  CRs, ArgoCD directory apps need `recurse: true` for subdirs.

## 2026-10-19 - kafka: Strimzi 1.3.0 operator + 3-node KRaft cluster

### Added

- `argocd/strimzi.yaml` — Strimzi operator pinned to 1.3.0 (chart from the
  upstream git repo; CRDs ship in the chart's crds/ dir).
- `argocd/kafka.yaml` + `kafka/cluster.yaml` — `lab` Kafka cluster: KRaft
  combined mode, 3 replicas (controller+broker each, no ZooKeeper), RF 3 /
  min.insync.replicas 2, 10Gi local-path PVs per broker, plain internal
  listener. portable_cluster blueprint slice 2.

## 2026-10-19 - garage: admin API bearer token (webui auth fix)

### Fixed

- The webui showed 0 nodes and auth errors on bucket operations. Two stacked
  causes: (1) the first image (khairul169:1.1.0) queries the removed v1 admin
  API — swapped to ghcr.io/davidschoefberger/garage-webui:2.5.2; (2) Garage
  v2's admin API REQUIRES a bearer token (403 otherwise) and none was
  configured — the in-pod CLI masks this because it authenticates via RPC with
  the rpc secret, not HTTP. Added hand-applied Secret `garage-admin-token`
  (never in this repo), injected as `GARAGE_ADMIN_TOKEN` into the daemon via
  the chart's `environment` passthrough and as `API_ADMIN_KEY` into the webui.
  Verified: GET /v2/GetClusterStatus with the token returns the live node
  list; the webui's /api/buckets lists lab-bucket.

## 2026-10-18 - garage-webui: admin UI for Garage

### Added

- `argocd/garage-webui.yaml` + `garage-webui/webui.yaml` —
  khairul169/garage-webui:1.1.0 (community admin UI), pointing at the Garage
  admin API via the headless service, exposed LAN-only at
  http://garage.vmk8.home.arpa (covered by the existing `*.vmk8` record).
  Security posture documented in the manifest: admin API is unauthenticated
  in-cluster (no admin_token set by the chart); harden via token +
  API_ADMIN_KEY_FILE + AUTH_USER_PASS if the dev posture stops being
  acceptable.

## 2026-10-18 - garage: rpcSecret churn incident — root cause, fix, prevention

### Fixed

- Restarted garage pods were permanently partitioned from the cluster (RPC
  handshake rejected both directions): an ArgoCD render of the chart without
  the live `lookup` value regenerated the rpcSecret in place, so pods that
  initialized before the churn held a different secret than restarted pods.
  Healed by rolling the last original pod; permanently prevented by switching
  to `garage.existingRpcSecret` (chart no longer renders the Secret; the live
  Secret is Prune=false protected). Full record:
  docs/garage-experiment-notes.md.

## 2026-10-18 - garage: experiment verified (layout, S3 round trip, node kill)

### Added

- `docs/garage-experiment-notes.md` — full run record: 3-node layout (256
  partitions, 3x replication), S3 write/read via aws-cli, node-kill test
  (object readable during outage, pod recreated on the same node with its
  local-path PVC), plus the gotchas hit (owner ≠ read/write in bucket allow,
  from-scratch image exec path, aws-cli entrypoint, Argo cosmetic OutOfSync).

## 2026-10-18 - garage: Garage v2.4.1 S3 object storage (portable_cluster experiment)

### Added

- `argocd/garage.yaml` + `garage/values.yaml` — upstream chart pinned to
  v2.4.1, 3-replica StatefulSet on `local-path` PVCs (meta 1Gi, data 20Gi),
  3x replication, Kubernetes-native peer discovery. rpcSecret auto-generated
  by the chart (helm-lookup-stable, nothing secret in this repo). First slice
  of the portable_cluster app-layer-sharded blueprint
  (infra/portable_cluster/README.md).

## 2026-10-18 - storage: local-path vs NFS benchmark

### Added

- `docs/storage-tier-benchmark.md` — measured local-path (ZFS/SATA vDisk) vs
  nfs-dataset (TrueNAS SSD pool): reads identical (host ARC), NFS ~4.7x faster
  on direct 4k random writes (ZFS sync/ZIL path dominates). Conclusion: the hot
  local tier pays off on dedicated NVMe nodes, not on the current hypervisor
  stack.

## 2026-10-18 - storage: local-path provisioner (hot local tier)

### Added

- `argocd/local-path.yaml` — rancher/local-path-provisioner v0.0.37, opt-in
  `local-path` StorageClass (not default) provisioning bind-mount PVs under
  `/opt/local-path-provisioner` per node. Zero per-volume daemons: the
  anti-Longhorn — replication stays in the app layer, local disk for hot data.

## 2026-10-18 - docs: NFS teardown-wedge issue record

### Added

- `docs/issues/2026_10_18_nfs_teardown_stuck_after_sigkill.md` — SIGKILL'd pod
  left the kubelet CSI mount dir populated, wedging pod/PVC/PV deletion
  (`directory not empty` retry loop). Remediation: empty the mount dir as root,
  kubelet's retry completes the teardown. Applies to prod too (same driver
  and StorageClass).

## 2026-10-18 - storage: NFS CSI driver + nfs-dataset StorageClass

### Added

- `argocd/nfs-csi.yaml` — upstream csi-driver-nfs chart pinned to v4.13.2,
  matching the prod cluster's driver version.
- `argocd/nfs-dataset.yaml` + `storage/nfs-dataset.yaml` — `nfs-dataset`
  StorageClass mirroring prod exactly: TrueNAS 192.168.0.10,
  `/mnt/ssd_pool/nfs_dataset`, hard NFSv4.1, Delete reclaim. Both clusters
  provision into the same share as distinct subdirectories.

## 2026-10-18 - dns: migrate dashboard host to home.arpa; pin pod DNS upstream

### Changed

- `traefik/values.yaml`: dashboard host `traefik.vmk8.local` → `traefik.vmk8.home.arpa`
  (`.local` is RFC 6762-reserved for mDNS and unusable from systemd-resolved hosts;
  prod CoreDNS now serves a consolidated `home.arpa` zone with `*.vmk8` → 192.168.0.95).

### Fixed

- Pod DNS for `home.arpa` names was nondeterministic (NXDOMAIN ~half the time):
  the node resolv.conf lists the router as a secondary resolver, and in-cluster
  CoreDNS's default random forward policy sent roughly half of pod queries to the
  router, which does not know the zone. Pinned `forward . 192.168.0.61` in the
  kube-system coredns ConfigMap (documented manual patch, see README — the
  configmap belongs to the microk8s dns addon, so it is not ArgoCD-managed).
  Verified 3/3 pod lookups after the rollout.

## 2026-10-18 - traefik: unblock first sync

### Fixed

- The traefik Application's first automated sync failed with
  `namespaces "ingress" not found` — the namespace does not pre-exist on a fresh
  cluster (it does on prod, which is why the mirrored manifest had no
  CreateNamespace). Added `CreateNamespace=true` to `argocd/traefik.yaml` and
  created the namespace by hand to clear the in-flight retry.

## 2026-10-18 - initial scaffold: app-of-apps, MetalLB, Traefik

### Added

- `bootstrap/apps.yaml` — app-of-apps root (the only manifest applied by hand).
- `argocd/metallb.yaml`, `argocd/traefik.yaml` — first two Applications.
- `metallb/` — upstream v0.15.2 manifest + kustomization (same pattern as prod) with the
  vmk8 dev pool `192.168.0.95-99`, disjoint from the prod pool (.60-.85) and the vmk8 node
  addresses (.90-.94).
- `traefik/values.yaml` — LAN-only dev ingress, chart pinned to 37.4.0 (parity with prod),
  LoadBalancer pinned to 192.168.0.95.
- `README.md`, `CLAUDE.md` — topology, LAN allocation table, DNS chain
  (hosts/pods resolve via prod CoreDNS at 192.168.0.61), dev/prod split rules.
- `docs/issues/2026_10_18_dhcp_ip_change_broke_dqlite.md` — root cause record for the
  IP-change incident that took down the pre-rebuild cluster and motivated the rebuild
  with router-assigned static addresses.
