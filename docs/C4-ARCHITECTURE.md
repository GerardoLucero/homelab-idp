# C4 Architecture — homelab-idp

This document describes the system architecture at 4 levels of detail: Context, Container, Component, and Code.

---

## Level 1: System Context

**What problem does this solve?**

A developer needs to deploy a microservice. Today:
- Call DevOps team (wait 3 days)
- Request infra ticket (wait 2 days)  
- Request CI/CD setup (wait 2 days)
- Request DNS + certs (wait 1 day)
- **Total: ~1-2 weeks before writing code**

**With this IDP:**
- Click "Create Service" in Backstage (1 minute)
- [Automated] repo created, pipeline configured, infrastructure provisioned, DNS set up
- **Total: 5 minutes, developer can start coding immediately**

```
┌─────────────────────────────────────────────────────────┐
│                    EXTERNAL USERS                       │
│  (Developers, DevOps, Security teams)                   │
└────────────────────────┬────────────────────────────────┘
                         │
                         │ SSH / HTTPS
                         │
        ┌────────────────┴────────────────┐
        │                                 │
   ┌────▼─────────┐            ┌────────▼──────┐
   │  Backstage   │            │   GitLab      │
   │ (IDP Portal) │            │ (Repository)  │
   └────┬─────────┘            └────────┬──────┘
        │                               │
        └───────────┬───────────────────┘
                    │
        ┌───────────▼───────────┐
        │ Kubernetes Cluster    │
        │ (Talos + ArgoCD)      │
        │ (All infrastructure)  │
        └───────────┬───────────┘
                    │
        ┌───────────▼───────────┐
        │ Internet              │
        │ (via Cloudflare)      │
        └───────────────────────┘
```

---

## Level 2: Container (Major System Components)

```
┌────────────────────────────────────────────────────────────────────┐
│                      KUBERNETES CLUSTER (Talos)                    │
│                                                                    │
│  ┌─────────────────────┐  ┌──────────────────┐                    │
│  │   Control Plane     │  │   Worker Nodes   │                    │
│  │  (3 replicas)       │  │   (3 replicas)   │                    │
│  │  - etcd             │  │                  │                    │
│  │  - API Server       │  │  - Workloads     │                    │
│  │  - Scheduler        │  │  - Storage       │                    │
│  └─────────────────────┘  └──────────────────┘                    │
│           │                       │                                │
│           └───────────────────────┘                                │
│                     │                                              │
├─────────────────────┼──────────────────────────────────────────────┤
│                     │                                              │
│  ┌──────────────────▼─────────────────┐                           │
│  │     ArgoCD (GitOps Engine)         │                           │
│  │  - Watches Git for changes         │                           │
│  │  - Syncs manifests to cluster      │                           │
│  └───────────────────────────────────┘                            │
│                                                                    │
│  ┌──────────────────────┬──────────────────────────────────────┐  │
│  │                      │                                      │  │
│  ▼                      ▼                                      ▼  │
│  ┌────────────┐   ┌──────────────┐   ┌────────────────────┐     │
│  │ Backstage  │   │ Monitoring   │   │ Security & Secrets │     │
│  │            │   │              │   │                    │     │
│  │- Portal    │   │- Prometheus  │   │- Vault             │     │
│  │- Catalog   │   │- Grafana     │   │- Kyverno           │     │
│  │- Auth      │   │- Alerting    │   │- External Secrets  │     │
│  └────────────┘   └──────────────┘   └────────────────────┘     │
│                                                                    │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐           │
│  │    GitLab    │   │    Harbor    │   │    Kong      │           │
│  │              │   │              │   │              │           │
│  │- SCM         │   │- Registry    │   │- API Gateway │           │
│  │- CI/CD       │   │- Cosign      │   │- Auth        │           │
│  │- Runners     │   │- SBOM        │   │- Rate Limit  │           │
│  └──────────────┘   └──────────────┘   └──────────────┘           │
│                                                                    │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐           │
│  │   Kafka      │   │ SonarQube    │   │ Authentik    │           │
│  │              │   │              │   │              │           │
│  │- Brokers     │   │- SAST        │   │- SSO/OIDC    │           │
│  │- Topics      │   │- Code Quality│   │- User Mgmt   │           │
│  │- CDC (Demo)  │   │- Reports     │   │- Groups      │           │
│  └──────────────┘   └──────────────┘   └──────────────┘           │
│                                                                    │
│  ┌──────────────┐   ┌──────────────┐                              │
│  │   Longhorn   │   │ cert-manager │                              │
│  │              │   │              │                              │
│  │- Storage     │   │- TLS Certs   │                              │
│  │- Replication │   │- Let's Encrypt                              │
│  │- Snapshots   │   │- Renewal     │                              │
│  └──────────────┘   └──────────────┘                              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
         │
         │ (All external traffic routed through)
         │
┌────────▼──────────────────────────────────────────────────────────┐
│              Cloudflare Tunnel (Zero-Trust)                       │
│  - No open ports on home network                                  │
│  - Automatic DDoS mitigation                                      │
│  - SSL/TLS termination                                            │
└────────────────────────────────────────────────────────────────────┘
```

---

## Level 3: Component (Key Container Internals)

### Backstage (IDP Portal)

```
┌─────────────────────────────────────────────────────┐
│            Backstage (IDP Portal)                   │
│                                                     │
│ ┌──────────────────────────────────────────────┐   │
│ │ Frontend (React)                             │   │
│ │ - Service Catalog UI                         │   │
│ │ - Kubernetes Viewer Plugin                   │   │
│ │ - ArgoCD Integration Plugin                  │   │
│ │ - SonarQube Scorecard Plugin                 │   │
│ └──────────────────────────────────────────────┘   │
│                    │                                │
│ ┌──────────────────▼──────────────────────────┐   │
│ │ Backend (Node.js)                            │   │
│ │ - Service Catalog API                        │   │
│ │ - LDAP/OIDC Auth (Authentik)                │   │
│ │ - K8s Metrics Aggregator                     │   │
│ │ - Scaffolder (Service Templates)             │   │
│ └──────────────────────────────────────────────┘   │
│                    │                                │
│ ┌──────────────────▼──────────────────────────┐   │
│ │ Database (PostgreSQL)                        │   │
│ │ - Service Metadata                           │   │
│ │ - User Profiles                              │   │
│ │ - Plugin State                               │   │
│ └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### DevSecOps Pipeline (git push → deployment)

```
┌─────────────────────────────────────────────────────────────┐
│  Developer: git push origin feature/new-service            │
└────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────▼─────────────────────┐
        │  GitLab CI Pipeline (Automated)           │
        │                                           │
        │  Stage 1: Secrets & Vulnerability Scan    │
        │  ├─ Gitleaks (secret scanning)            │
        │  ├─ SAST: SonarQube                       │
        │  ├─ Container: Trivy (CVE scan)           │
        │  └─ Signature: Cosign (sign image)        │
        │                                           │
        │  Stage 2: Build & Push                    │
        │  ├─ Build OCI image                       │
        │  ├─ Push to Harbor                        │
        │  └─ Generate SBOM (Cosign attestation)    │
        │                                           │
        │  Stage 3: Policy & Compliance              │
        │  ├─ Verify image signature                │
        │  ├─ Run supply-chain security checks      │
        │  └─ Log to audit trail (Vault)            │
        │                                           │
        │  Stage 4: GitOps Sync                      │
        │  └─ Update deployment manifests in Git    │
        └──────────────────┬──────────────────────┘
                           │
        ┌──────────────────▼──────────────────┐
        │ ArgoCD (Watches Git)                │
        │ - Detects manifest changes          │
        │ - Applies to Kubernetes cluster     │
        │ - Webhook notification              │
        └──────────────────┬──────────────────┘
                           │
        ┌──────────────────▼──────────────────┐
        │ Kyverno (Policy Enforcement)        │
        │ - Validates: image registry?        │
        │ - Validates: resource quotas?       │
        │ - Validates: network policies?      │
        │ - Rejects if doesn't comply         │
        └──────────────────┬──────────────────┘
                           │
        ┌──────────────────▼──────────────────┐
        │ Pod Deployed to Cluster             │
        │ - Monitoring begins (Prometheus)    │
        │ - Logs sent (Grafana Loki)          │
        │ - Accessible via Kong Gateway       │
        └──────────────────────────────────────┘
```

---

## Level 4: Code (Specifics — see manifests/)

Key codebases:

- **manifests/** — Helm values + Kustomization overlays per service
- **docs/getting-started.md** — Step-by-step deployment
- **Talos machine config** — Node bootstrap (API-driven)
- **GitHub Actions** — Local testing before GitLab CI

---

## Data Flow Summary

1. **Developer**: Pushes code to GitLab
2. **GitLab CI**: Runs security scans, builds image, pushes to Harbor
3. **ArgoCD**: Watches Git, detects manifest changes, applies to cluster
4. **Kyverno**: Enforces policies at admission time
5. **Kubernetes**: Schedules pod, starts monitoring
6. **Backstage**: Portal shows service status, metrics, deployment history

---

## Security Boundaries

```
┌─────────────────────────────────────────┐
│  UNTRUSTED (Internet)                   │
└──────────────┬──────────────────────────┘
               │
        [Cloudflare Tunnel]
        [TLS Termination]
        [DDoS Protection]
               │
┌──────────────▼──────────────────────────┐
│  SEMI-TRUSTED (Kong + Authentik)        │
│  - Rate limiting                        │
│  - OIDC authentication required         │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│  TRUSTED (Inside Cluster)               │
│  - All service-to-service requires:     │
│    * mTLS (Istio/cert-manager)          │
│    * RBAC (Kubernetes)                  │
│    * Network policies (Kyverno)         │
│  - Secrets never in ConfigMaps          │
│  - Secrets via Vault + ExternalSecrets  │
└──────────────────────────────────────────┘
```

---

## Next Steps

- For deployment: see [docs/getting-started.md](./getting-started.md)
- For infrastructure code: see [../manifests/](../manifests/)
- For GitOps workflow: see [docs/gitops-workflow.md](./gitops-workflow.md) (coming soon)
