# SIGKILL'd pod leaves the NFS mount dir populated, wedging kubelet teardown and blocking pod/PVC/PV deletion

**Date:** 2026-10-18  
**Component:** csi-driver-nfs v4.13.2 node plugin teardown (`NodeUnpublishVolume` / `Unmounter.TearDownAt`), MicroK8s kubelet volume reconciler  
**Severity:** Medium — single pod/PVC/PV stuck `Terminating`; no data loss, but deletion of the workload is blocked until the mount dir is emptied by hand

---

## Observed symptom

After the NFS smoke test, deleting a pod that had just written to an `nfs-dataset` PVC
left all three objects stuck: pod `Error`/`Terminating` (no finalizers),
PVC `Terminating` holding `kubernetes.io/pvc-protection`, PV `Bound`. Kubelet retried
`UnmountVolume.TearDown` every ~2 minutes, each time failing with:

```
Unmounter.TearDownAt failed to clean mount dir
[.../volumes/kubernetes.io~csi/<pvc>/mount]:
failed to remove dir [...]: directory not empty
```

The pod had been killed with SIGKILL (exit 137) seconds after writing `probe.txt`.

---

## Root cause

### The kubelet-side mount dir still held the NFS view at teardown time

`TearDownAt` removes the kubelet mount directory; the removal failed because the
directory was not empty — it still contained `probe.txt` (the file the killed pod had
written). Verified directly: `sudo ls` on the mount dir showed only `probe.txt`;
after emptying it, the next kubelet retry completed and the pod, PVC and PV all
deleted within one retry cycle.

⚠ unverified: the exact reason the NFS view was still present at `TearDownAt` time
(unmount ordering race after the hard kill, or an NFS silly-rename `.nfs*` residue
blocking the unmount — both fit the log; the dir listing showing plain `probe.txt`
favors the former). What matters operationally was verified: emptying the mount dir
unblocks the retry loop.

---

## Troubleshooting steps taken

1. **Checked PVC/PV finalizers and pod state** — ruled out finalizer-based stalls on
   the PVC/PV themselves; the pod had none either, but carried a deletionTimestamp
   for 6+ minutes.
2. **Read the node's kubelite journal** (daniel is in `adm`) — found the
   `UnmountVolume.TearDown failed ... directory not empty` loop with exponential
   retry backoff; identified the CSI NFS teardown as the blocker, not the API server
   or the external-provisioner.
3. **Attempted cleanup with a dotfile glob** (`rm ... /.[!.]*`) — no effect; the
   leftover was a regular file, not a `.nfs*` orphan.
4. **Listed the mount dir as root** — `probe.txt` (23 bytes) was the only content.
5. **Emptied the dir with `find <mount> -mindepth 1 -delete`** — next kubelet retry
   (within ~2 min) succeeded; pod, PVC and PV all reached NotFound.

---

## Fix

### vmk8node3 — emptied the kubelet CSI mount dir for the stuck pod

```bash
M=/var/snap/microk8s/common/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<pv-name>/mount
sudo find $M -mindepth 1 -delete    # empty it, let kubelet's retry finish the teardown
```

No git changes; this is an operational remediation. Prevention: avoid `kubectl delete
pod` (which SIGKILLs after grace) on pods mid-write to NFS volumes in throwaway tests —
or expect to run this cleanup. The same wedging applies on the prod cluster, which
runs the identical driver and StorageClass.

---

## Files changed

- none (operational fix; recorded here for the next session that hits it)
