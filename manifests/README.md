# Manifests

Raw Kubernetes manifests managed by ArgoCD Applications defined in `apps/`.

Each subdirectory is referenced by an ArgoCD Application as a `path` source.

## Structure

- `gateway/` — Gateway API infrastructure: Cilium LB-IPAM pool, L2 announcement policy, shared HTTP Gateway, and per-app HTTPRoutes.
