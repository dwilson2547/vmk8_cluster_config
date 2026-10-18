# vmk8_cluster_config

ArgoCD application definitions and manifests for the vmk8 dev cluster. Mirrors
`infra/cluster-config` (prod); read that repo's CLAUDE.md for the full pattern.

## Rules

- Every ArgoCD `Application` lives in `argocd/` here, synced by the app-of-apps root
  `bootstrap/apps.yaml` (the only manifest applied by hand). Adding or editing an Application
  is just a git push to `main`.
- Never `kubectl apply` anything ArgoCD-managed. Git push is the deployment mechanism.
- Never `microk8s enable`/`disable` the `metallb` or `traefik` addons: ArgoCD owns those
  components here, and the addon lifecycle would fight it (and can delete LB IPs).
- The MetalLB pool must stay disjoint from prod's 192.168.0.60-85 and from the node addresses
  192.168.0.90-94. Both clusters announce L2 on the same LAN.
- DNS authority is prod's CoreDNS (192.168.0.61, zone files in `infra/cluster-config/dns/`).
  Hosts and pods here resolve through it; to publish a vmk8 ingress hostname, repoint its A
  record to 192.168.0.95 in the prod repo and bump the serial.
- Never commit secrets or kubeconfigs; apply secrets by hand.
- Every change to this repo gets a dated `CHANGELOG.md` entry. Non-negotiable.

## Dev/prod split

Dev variants of services deploy here (their own databases, never prod's); prod versions stay
in `infra/cluster-config`. When a dev app has a Helm chart in its own repo, its Application
manifest still lives in `argocd/` here, pointed at the chart's dev branch/values.
