# GitOps Repository

## How It Works

The [infra repo](https://github.com/barnes-c/infra) uses OpenTofu to bootstrap the Talos cluster, install Cilium CNI,
and deploy ArgoCD. ArgoCD then takes over and syncs everything in this repository automatically.

```txt
OpenTofu (infra repo)          ArgoCD (this repo)
─────────────────────          ──────────────────
Talos cluster         ──►      apps-root Application
Cilium (bootstrap)             watches apps/ directory
ArgoCD (bootstrap)             syncs all Applications
```

## Repository Structure

```bash
apps/                              # ArgoCD Application definitions
├── cilium.yaml                    # CNI + Gateway API controller
├── gateway-api-crds.yaml          # Gateway API CRDs (from upstream v1.4.1)
├── gateway.yaml                   # Gateway, HTTPRoutes, LB pool, L2 policy
├── cert-manager.yaml              # TLS certificate management (Helm)
├── cert-manager-config.yaml       # ClusterIssuer + Certificate resources
├── longhorn.yaml                  # Distributed storage
├── argocd.yaml                    # ArgoCD (self-managed)
├── monitoring.yaml                # Prometheus + Grafana
└── other-apps.yaml

manifests/                         # Raw Kubernetes manifests
├── gateway-api-crds/              # Vendored Gateway API CRDs
│   └── standard-install.yaml
├── cert-manager/                  # cert-manager custom resources
│   ├── clusterissuer.yaml         # Let's Encrypt ClusterIssuer (Cloudflare DNS-01)
│   └── certificate.yaml           # Wildcard cert for *.barnes.biz
└── gateway/                       # Gateway API resources
    ├── pool.yaml                  # CiliumLoadBalancerIPPool (192.168.1.201-210)
    ├── l2-policy.yaml             # CiliumL2AnnouncementPolicy
    ├── gateway.yaml               # Shared Gateway (HTTP :80 + HTTPS :443)
    ├── other-routes.yaml          # e.g. argocd.barnes.biz → ArgoCD UI
```

## Deployment Order

Applications deploy in order via [sync waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/):

| Wave | Application        | Purpose                    |
|------|--------------------|----------------------------|
| -3   | Cilium             | CNI / networking           |
| -2   | Gateway API CRDs   | Gateway API definitions    |
| -2   | Longhorn           | Storage                    |
| -1   | Cert-Manager       | TLS certificates (Helm)    |
| -1   | Gateway            | Gateway API infrastructure |
| 0    | Cert-Manager Config| ClusterIssuer + Certs      |
| 0    | ArgoCD             | GitOps controller          |
| 0    | Monitoring         | Prometheus + Grafana       |
| 1    | Other apps         | Application workloads      |

## Secrets

Secrets are encrypted with [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) and stored
as `SealedSecret` resources in Git. The controller (deployed at sync wave `-1`) decrypts them in-cluster.

To add or rotate a secret:

```bash
kubectl create secret generic <name> \
  --namespace <namespace> \
  --from-literal=<key>=<value> \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace sealed-secrets -o yaml \
  > manifests/<app>/sealed-<name>.yaml
```

## Networking

Traffic is routed via [Cilium Gateway API](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/):

- Cilium assigns LoadBalancer IPs from `192.168.1.201-210` and announces them on the LAN via L2/ARP
- A shared `Gateway` listens on HTTP (:80) and HTTPS (:443)
- `HTTPRoute` resources route traffic by hostname to backend services
- DNS for `*.barnes.biz` points to `192.168.1.201`

### Adding a New Route

Create an HTTPRoute in `manifests/gateway/`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: my-app-namespace
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: main
      namespace: gateway
  hostnames:
    - my-app.barnes.biz
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - group: ""
          kind: Service
          name: my-app-service
          port: 80
          weight: 1
```

## Adding a New Application

1. Create a YAML file in `apps/` with an ArgoCD Application definition
2. Set the appropriate `sync-wave` annotation
3. Push — ArgoCD will pick it up automatically

## Dependencies

- [infra](https://github.com/barnes-c/infra) — OpenTofu code for Talos cluster bootstrap
- Dependency updates are managed by [Renovate](https://github.com/renovatebot/renovate)
