---
title: NFS teardown wedge after SIGKILL on vmk8
date: 2026-10-18
tags: k8s,nfs,csi
---

SIGKILLing a pod mid-write on an nfs-dataset volume can leave the kubelet CSI mount dir populated, wedging pod/PVC/PV deletion in a directory-not-empty retry loop. Fix: sudo find <kubelet pods/<uid>/volumes/kubernetes.io~csi/<pv>/mount> -mindepth 1 -delete, then let kubelet's ~2min retry finish. Applies to prod (same driver v4.13.2 + StorageClass). See: docs/issues/2026_10_18_nfs_teardown_stuck_after_sigkill.md
