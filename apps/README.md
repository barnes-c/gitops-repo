# Apps

ArgoCD Application definitions. Each YAML file defines a Helm chart or manifest source that ArgoCD deploys and manages.

The `apps-root` Application (created by OpenTofu in the [infra repo](https://github.com/barnes-c/infra)) watches this directory and automatically syncs any Application added here.

## Sync Waves

Applications are deployed in order using `argocd.argoproj.io/sync-wave` annotations:

| Wave | Application  | Purpose                   |
|------|-------------|----------------------------|
| -3   | Cilium      | CNI / networking           |
| -2   | Longhorn    | Storage                    |
| -1   | Cert-Manager| TLS certificates           |
| -1   | Gateway     | Gateway API infrastructure |
| 0    | ArgoCD      | GitOps controller          |
| 0    | Monitoring  | Prometheus + Grafana       |
| 1    | Immich      | Application workloads      |
