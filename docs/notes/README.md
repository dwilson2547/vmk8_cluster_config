# Notes index

- [Garage slice of portable_cluster deployed and verified](garage-slice-of-portable-cluster-deployed-and-verified.md) — Garage v2.4.1 on vmk8 (ArgoCD app, upstream chart from deuxfleurs-org/garage): 3 replicas on local-path PVCs,… `s3,garage,portable-cluster`
- [NFS teardown wedge after SIGKILL on vmk8](nfs-teardown-wedge-after-sigkill-on-vmk8.md) — SIGKILLing a pod mid-write on an nfs-dataset volume can leave the kubelet CSI mount dir populated, wedging po… `k8s,nfs,csi`
- [vmk8 dev cluster rebuilt and under GitOps](vmk8-dev-cluster-rebuilt-and-under-gitops.md) — The vmk8 MicroK8s cluster (5 VMs on the Epyc box, 192.168.0.90-94, router-assigned) was rebuilt after a DHCP … `k8s,argocd,infra`
