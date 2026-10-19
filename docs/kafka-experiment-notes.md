# Kafka experiment notes (portable_cluster slice 2)

**Date:** 2026-10-19, vmk8 cluster, Strimzi 1.3.0 / Kafka 4.3.1 (KRaft,
combined controller+broker pool of 3 on `local-path` PVs, 10Gi each).

## Results (all verified)

- **Cluster Ready**: 3 brokers on 3 different nodes, KRaft quorum formed
  (voters 0/1/2), entity operator (topic + user) running.
- **Topic as code**: `kafka/topics/lab-test-topic.yaml` (6 partitions, RF 3,
  min.insync.replicas 2, 1h retention) applied via GitOps.
- **Partition spread**: leaders distributed across all three brokers (P2/P4 →
  broker 0, P0/P5 → broker 1, P1/P3 → broker 2), full ISR everywhere.
- **Produce/consume**: 100 messages produced, consumed back in order.
- **Broker-kill test**: deleted `lab-mixed-1`; **produced 20 more messages
  during the outage** (min ISR 2 held on the survivors), leaders failed over
  off the dead broker (P0/P5 moved to broker 2), pod recreated on the same
  node with its PVC, ISR recovered. Final consume from beginning:
  **120/120 messages, zero loss**.

## Gotchas hit (for future sessions)

- **Strimzi 1.x serves `kafkas.kafka.strimzi.io` at v1 only** — v1beta2 is
  gone for the Kafka kind (KafkaTopic is also v1). Using v1beta2 fails with a
  confusing "could not discover server resources" sync error.
- **Strimzi 1.x requires KafkaNodePool CRs** — `spec.kafka.replicas` no longer
  exists ("field not declared in schema"). Combined mode = one pool with
  `roles: [controller, broker]`; storage + resources live on the pool.
- **ArgoCD directory apps do NOT recurse into subdirectories by default** —
  `kafka/topics/` was silently invisible until `directory.recurse: true` was
  set on the Application source. Check other apps before nesting manifests.
- **ArgoCD repo-server had a transient `DeadlineExceeded` render blip and a
  stale-manifest episode** — a rollout restart of `deploy/argocd-repo-server`
  clears render caches if a Synced app renders old manifests.
- Kafka version pinning comes from the CR (`version: 4.3.1`,
  `metadataVersion: 4.3-IV0` per the upstream example for 1.3.0).
