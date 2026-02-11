# Manifests

Raw Kubernetes manifests managed by ArgoCD Applications defined in `apps/`.

Each subdirectory is referenced by an ArgoCD Application as a `path` source.

## Structure

- `barnes-biz/` — Personal website deployment, cloudflared tunnel, and sealed secret.
- `cert-manager/` — ClusterIssuer, TLS certificate, and sealed Cloudflare API token.
- `gateway/` — Gateway API infrastructure: Cilium LB-IPAM pool, L2 announcement policy,
shared HTTP Gateway, and per-app HTTPRoutes.
