<div align="center">

# Procurement Platform — GitOps

[![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?logo=argo&logoColor=white)](https://argoproj.github.io/cd)
[![Helm](https://img.shields.io/badge/Helm-3-0F1689?logo=helm&logoColor=white)](https://helm.sh)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.30-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![AWS EKS](https://img.shields.io/badge/AWS-EKS-FF9900?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/eks)
[![Kgateway](https://img.shields.io/badge/Kgateway-Gateway%20API-6E48AA)](https://kgateway.dev)

**Helm chart + ArgoCD Application manifests** for the Procurement Platform.

ArgoCD watches this repository — `develop` branch for the dev environment, `main` for prod.
No CI/CD pipeline ever touches the cluster directly; **all changes flow through this repo**.

[Organization](https://github.com/ProcurementPlatform) · [App Repo](https://github.com/ProcurementPlatform/procurement-platform-app) · [Infrastructure Repo](https://github.com/ProcurementPlatform/procurement-platform-infra)

</div>

---

## Cloud Infrastructure & GitOps Flow

![Cloud Infrastructure Architecture](docs/cloud-architecture.png)

---

## GitOps Deployment Flow

```
Developer pushes code to procurement-platform-app
              │
              ▼
   build.yml passes all stages
   (lint → SonarCloud + Snyk → Docker build → Trivy → ECR push)
              │
              ▼
   deploy.yml runs:
   1. Downloads image tag artifact from Build run
   2. Reads IRSA role ARNs from Terraform remote state
   3. Bumps image.tag + IRSA annotations in
      helm/procurement-platform/values-dev.yaml   (develop branch)
      helm/procurement-platform/values-prod.yaml  (main branch)
   4. Commits and pushes to THIS repo
              │
              ▼
   ArgoCD detects the values file change
   (polling interval or webhook)
              │
              ▼
   ArgoCD syncs the Helm release to EKS
   develop → procurement-dev namespace
   main    → procurement-prod namespace
```

---

## Repository Structure

```
procurement-platform-gitops/
├── applications/
│   ├── app-of-apps.yaml                    # Root ArgoCD Application (App of Apps pattern)
│   └── environments/
│       ├── dev/platform.yaml               # ArgoCD Application → dev namespace, watches develop branch
│       └── prod/platform.yaml              # ArgoCD Application → prod namespace, watches main branch
├── helm/
│   └── procurement-platform/               # Umbrella Helm chart
│       ├── Chart.yaml                      # Chart metadata and subchart dependencies
│       ├── values.yaml                     # Global defaults (ports, resource limits, probes)
│       ├── values-dev.yaml                 # Dev overrides: image tags + ECR URLs + IRSA ARNs
│       ├── values-prod.yaml                # Prod overrides: image tags + ECR URLs + IRSA ARNs
│       ├── templates/
│       │   ├── gateway.yaml                # Shared Kgateway Gateway resource (one per environment)
│       │   ├── gatewayparameters.yaml      # Kgateway parameters (NLB config, TLS)
│       │   └── networkpolicies.yaml        # Network policies between namespaces
│       └── charts/                         # Per-service subcharts (one per microservice)
│           ├── frontend/
│           ├── identity-service/
│           ├── procurement-service/
│           ├── finance-service/
│           ├── document-service/
│           └── ai-service/
│               └── templates/
│                   ├── deployment.yaml      # Deployment with resource limits and probes
│                   ├── service.yaml         # ClusterIP service
│                   ├── configmap.yaml       # Non-sensitive runtime configuration
│                   ├── serviceaccount.yaml  # ServiceAccount with IRSA role ARN annotation
│                   ├── httproute.yaml       # Kgateway HTTPRoute (path-based routing)
│                   ├── hpa.yaml             # Horizontal Pod Autoscaler
│                   ├── pdb.yaml             # Pod Disruption Budget (prod: minAvailable=1)
│                   └── servicemonitor.yaml  # Prometheus ServiceMonitor for Grafana
└── k8s/
    ├── dev/                                # Static reference manifests (dev)
    │   ├── namespace.yaml
    │   ├── gateway/gateway.yaml
    │   └── {service}/                      # deployment, service, configmap, serviceaccount, httproute
    └── prod/                               # Static reference manifests (prod)
        ├── namespace.yaml
        ├── gateway/gateway.yaml
        └── {service}/
```

> **Note on `k8s/` directories:** These manifests are **not** applied by ArgoCD. They are static reference manifests that mirror what the Helm chart renders, included for documentation and evaluator review. Account ID, S3 bucket, domain, and ACM cert ARN are placeholder values.

---

## ArgoCD — App of Apps Pattern

```
app-of-apps.yaml  (root Application, manually applied once to bootstrap)
        │
        ├── environments/dev/platform.yaml
        │       └── watches helm/ on the develop branch of this repo
        │           applies to: procurement-dev namespace
        │
        └── environments/prod/platform.yaml
                └── watches helm/ on the main branch of this repo
                    applies to: procurement-prod namespace
```

ArgoCD is installed by `procurement-platform-infra/scripts/bootstrap-cluster.sh`. After ArgoCD is running, applying the App-of-Apps manifest is the only manual step — from that point forward, all deployments are fully automated through this repo.

---

## Routing — Kgateway (Gateway API)

Each environment has **one shared Gateway** backed by a single Envoy proxy and one AWS Network Load Balancer:

```
Internet
    │  HTTPS
    ▼
CloudFront + WAF
    │
    ▼
NLB (provisioned by Kgateway via AWS Load Balancer Controller)
    │
    ▼  Kgateway (Envoy) — longest-prefix match routing
    ├── /api/identity/*        →  identity-service:5001
    ├── /api/procurement/*     →  procurement-service:5003
    ├── /api/finance/*         →  finance-service:5002
    ├── /api/documents/*       →  document-service:5004
    ├── /api/ai/*              →  ai-service:5006
    └── /*                     →  frontend (React SPA catch-all)
```

**Key design decisions:**
- TLS terminates at the NLB using the ACM certificate (`global.acmCertArn` in `values.yaml`) — Envoy listeners are plain HTTP internally
- Gateway API's longest-prefix match ensures `/api/*` routes are never shadowed by the frontend `/` catch-all — no `group.order` needed
- Kgateway (Envoy-based) replaces per-service ALB Ingress resources with a single shared Gateway
- Kgateway and the Gateway API CRDs are installed cluster-wide by the infra bootstrap script

---

## Image Versioning Strategy

| Branch | Environment | Image Tag | Example |
|---|---|---|---|
| `develop` | dev | Git commit SHA (short) | `a3f92c1` |
| `main` | prod | Semantic version | `v1.2.0` |

Both `image.repository` and `image.tag` are bumped by the app repo's `deploy.yml` on every successful build and push. IRSA role ARN annotations are also refreshed from Terraform state on each deploy.

---

## Secrets Management

**No secrets are stored in this repository** — not in ConfigMaps, not in Helm values, not in environment variables.

All runtime secrets are pulled from **AWS Secrets Manager** at pod startup via **IRSA** (IAM Roles for Service Accounts):

```
Pod boots
  │
  └── Kubernetes ServiceAccount has IRSA annotation:
      eks.amazonaws.com/role-arn: arn:aws:iam::<account>:role/<service>-irsa-role
        │
        ▼
      OIDC token exchanged for temporary AWS credentials (valid 24h, auto-rotated)
        │
        ▼
      Service reads secrets from:
      aws secretsmanager get-secret-value --secret-id procurement/{env}/{service}/*
```

---

## Monitoring Integration

Each service subchart includes a `ServiceMonitor` resource that is picked up by the **kube-prometheus-stack** (Prometheus + Grafana + Alertmanager) running in the monitoring namespace. Services expose a `/metrics` endpoint (Prometheus client format) for scraping.

Grafana dashboards for the platform are defined in the infra repo at `scripts/grafana-dashboards/procurement-app.json`.

---

## Related Repositories

| Repository | Description |
|---|---|
| [procurement-platform-app](https://github.com/ProcurementPlatform/procurement-platform-app) | Backend microservices + React frontend + build/deploy CI pipeline |
| [procurement-platform-infra](https://github.com/ProcurementPlatform/procurement-platform-infra) | Terraform — VPC, EKS, DynamoDB, S3, CloudFront, WAF, IAM/IRSA, KMS, CloudWatch |

---

## Author & Contributors

| Name | Role | GitHub |
|---|---|---|
| **Vinay Kumar Kondoju** | Owner & Lead Developer | [@KONDOJUVINAYKUMAR08](https://github.com/KONDOJUVINAYKUMAR08) |

---

<div align="center">

**Procurement Platform** — Final Capstone Project

GitOps with ArgoCD · Helm · Kubernetes · Kgateway · AWS EKS

</div>
