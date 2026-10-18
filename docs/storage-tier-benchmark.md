# Storage tier benchmark: local-path vs nfs-dataset

**Date:** 2026-10-18, single run, 10s per job, O_DIRECT (`--direct=1`), iodepth 32,
4k random + 1M sequential, one pod on vmk8snode3, fio from alpine.

## Environment

- `local-path`: bind-mount PV on the VM root disk — a ZFS-backed vDisk on SATA
  SSD on the Epyc hypervisor host.
- `nfs-dataset`: hard NFSv4.1 mount of the TrueNAS (192.168.0.10)
  `/mnt/ssd_pool/nfs_dataset` SSD pool.

## Results

| Job | local-path | nfs-dataset |
|---|---|---|
| 4k random read | 12.1k IOPS / 47.4 MiB/s | 11.8k IOPS / 45.9 MiB/s |
| 4k random write | **440 IOPS / 1.8 MiB/s** | **2060 IOPS / 8.2 MiB/s** |
| 1M sequential read | 3296 MiB/s | 3309 MiB/s |
| 1M sequential write | 15.5 MiB/s | 16.4 MiB/s |

## Interpretation

- **Local did NOT beat NFS** on this hardware stack. Random and sequential reads
  are identical (both effectively served from ZFS ARC host RAM, so neither the
  virtio hop nor the network mattered). On direct random writes NFS was ~4.7x
  faster than local.
- The dominating factor is ZFS sync-write semantics, not local-vs-network: the
  guest's O_DIRECT writes reach the Epyc's ZFS pool as sync commits through a
  SATA ZIL with no SLOG device, which crushes them (440 IOPS). The sequential
  direct writes collapse to ~15 MiB/s on both tiers for the same per-commit
  reason. ⚠ unverified: whether TrueNAS's ssd_pool has `sync=disabled` or simply
  a healthier ZIL setup — its 4k write advantage points at one of the two.
- **Caveats:** this is the worst-case sync-heavy pattern, not representative of
  application writes (Postgres/Kafka batch their own fsyncs rather than issuing
  O_DIRECT per 4k; buffered writes would look very different). Single run, one
  node, short windows — indicative, not rigorous.

## Design implication

The hot-local-tier plan stands, but the payoff comes from **dedicated nodes with
NVMe and a non-ZFS local filesystem (ext4/XFS)**, where O_DIRECT is truly direct
and there is no hypervisor or ZIL in the path. Expecting the current Epyc/ZFS
vDisk tier to outperform NFS is not supported by these numbers; the levers on the
existing box would be a SLOG device or relaxed sync settings, both of which are
hypervisor-side changes outside this cluster's control.
