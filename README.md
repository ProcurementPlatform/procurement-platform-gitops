# procurement-platform-gitops

Helm chart + ArgoCD Application manifests for the Procurement Platform. ArgoCD (installed
manually via `procurement-platform-infra/scripts/bootstrap-cluster.sh`) watches this repo only —
the `develop` branch for the `dev` environment, `main` for `prod`. No CI/CD pipeline ever deploys
directly to the cluster; the only path in is a sync from this repo.

## Structure

- `helm/procurement-platform/` — the live, ArgoCD-synced deployment path. Umbrella chart with one
  global `values.yaml` (per-service defaults consolidated here) plus `values-dev.yaml` /
  `values-prod.yaml` for environment-specific image tags and IRSA role ARNs, injected by the app
  repo's `deploy.yml` at deploy time. ConfigMaps live inside the Helm templates for non-sensitive
  configuration only — secrets always come from Secrets Manager via IRSA at runtime, never from a
  ConfigMap or Helm value.
- `k8s/dev/` and `k8s/prod/` — static reference manifests, one folder per service (deployment,
  service, configmap, serviceaccount, httproute) plus `namespace.yaml` and a `gateway/` folder
  holding the shared Gateway. These are **not** applied by ArgoCD; they exist to satisfy the
  literal manifest checklist for documentation/evaluator review and mirror what Helm renders.
  Account ID, S3 bucket name, domain, and ACM cert ARN are placeholders — substitute real values
  only when applying manually.
- `applications/` — the ArgoCD `Application` / App-of-Apps manifests that point back at the
  `helm/` path in this same repo.

## Routing (Kgateway / Gateway API)

Routing uses **Kgateway** (Envoy-based Gateway API implementation), not Kubernetes Ingress. Each
environment gets **one shared `Gateway`** (rendered by the umbrella chart) which Kgateway backs
with a single Envoy proxy and one NLB; every service attaches to it via its own `HTTPRoute`
(replacing the old per-service ALB `Ingress`). TLS terminates at the NLB using the ACM cert in
`global.acmCertArn`, so the Envoy listeners are plain HTTP. Gateway API ranks routes by
longest-prefix match, so the frontend `/` catch-all never shadows the `/api/*` routes — no
`group.order` needed. Kgateway and the Gateway API CRDs are installed cluster-wide by
`procurement-platform-infra` (`modules/eks-addons`). ArgoCD's own UI is exposed by a separate
LoadBalancer and is unaffected by this routing layer.

## Images & versioning

- **dev** (`develop` → `values-dev.yaml`): images tagged with the **git commit SHA**.
- **prod** (`main` → `values-prod.yaml`): images tagged with the **release version** (git tag, e.g.
  `v1.2.0`).

Both `image.repository`/`image.tag` and IRSA annotations are bumped by the app repo's `deploy.yml`.

## Related repos

- [procurement-platform-app](https://github.com/ProcurementPlatform/procurement-platform-app) — backend/frontend source + build/deploy CI
- [procurement-platform-infra](https://github.com/ProcurementPlatform/procurement-platform-infra) — Terraform infra + cluster add-ons
