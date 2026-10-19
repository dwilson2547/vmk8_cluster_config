# Notes index

- [CockroachDB helm chart init Job deadlocks under ArgoCD](cockroachdb-helm-chart-init-job-deadlocks-under-argocd.md) — The official cockroachdb helm chart's init Job carries helm.sh/hook: post-install; ArgoCD runs it as a PostSy… `cockroachdb,helm,argocd`
- [Garage slice of portable_cluster deployed and verified](garage-slice-of-portable-cluster-deployed-and-verified.md) — Garage v2.4.1 on vmk8 (ArgoCD app, upstream chart from deuxfleurs-org/garage): 3 replicas on local-path PVCs,… `s3,garage,portable-cluster`
- [helm lookup secrets are not render-stable under ArgoCD](helm-lookup-secrets-are-not-render-stable-under-argocd.md) — Charts that generate secrets via helm lookup + randHex fallback will silently regenerate the value on any ren… `helm,argocd,gitops,secrets`
- [NFS teardown wedge after SIGKILL on vmk8](nfs-teardown-wedge-after-sigkill-on-vmk8.md) — SIGKILLing a pod mid-write on an nfs-dataset volume can leave the kubelet CSI mount dir populated, wedging po… `k8s,nfs,csi`
- [Strimzi 1.x on k8s: v1 CRDs, node pools, argocd recurse](strimzi-1-x-on-k8s-v1-crds-node-pools-argocd-recurse.md) — Strimzi 1.x: Kafka and KafkaTopic CRDs serve v1 only (v1beta2 gone); broker topology must use KafkaNodePool C… `kafka,strimzi,argocd`
- [vmk8 dev cluster rebuilt and under GitOps](vmk8-dev-cluster-rebuilt-and-under-gitops.md) — The vmk8 MicroK8s cluster (5 VMs on the Epyc box, 192.168.0.90-94, router-assigned) was rebuilt after a DHCP … `k8s,argocd,infra`
