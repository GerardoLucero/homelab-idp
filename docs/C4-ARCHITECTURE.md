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

```mermaid
graph TB
    subgraph External["External Users"]
        DEV["Developers"]
        DEVOPS["DevOps Engineers"]
        SEC["Security Teams"]
    end
    
    subgraph IDP["Internal Developer Platform"]
        BACKSTAGE["Backstage<br/>(Portal + Catalog)"]
        GITLAB["GitLab<br/>(Repository)"]
    end
    
    subgraph CLUSTER["Kubernetes Cluster"]
        K8S["Talos K8s<br/>(6 nodes)"]
    end
    
    subgraph INTERNET["Internet"]
        CF["Cloudflare Tunnel<br/>(Zero-Trust)"]
    end
    
    DEV -->|SSH/HTTPS| BACKSTAGE
    DEVOPS -->|SSH/HTTPS| GITLAB
    SEC -->|SSH/HTTPS| BACKSTAGE
    
    BACKSTAGE -->|Git Push| GITLAB
    GITLAB -->|ArgoCD Sync| K8S
    K8S -->|Cloudflare| CF
    
    style DEV fill:#e1f5ff
    style DEVOPS fill:#f3e5f5
    style SEC fill:#fce4ec
    style BACKSTAGE fill:#81c784
    style GITLAB fill:#64b5f6
    style K8S fill:#ffb74d
    style CF fill:#ff8a65
```

---

## Level 2: Container (Major System Components)

The entire system runs within a single Kubernetes cluster on Talos Linux.

```mermaid
graph TB
    subgraph TALOS["Talos Linux Cluster (6 nodes)"]
        subgraph CP["Control Plane (3 nodes)"]
            ETCD["etcd<br/>(Distributed DB)"]
            APISERVER["API Server"]
            SCHEDULER["Scheduler"]
        end
        
        subgraph WORKERS["Worker Nodes (3 nodes)"]
            WORKLOADS["Pod Scheduling"]
            STORAGE["Block Storage<br/>(Longhorn)"]
        end
        
        subgraph GITOPS["GitOps Engine"]
            ARGOCD["ArgoCD<br/>(Sync State)"]
        end
        
        subgraph SERVICES["Core Services"]
            BACKSTAGE["Backstage<br/>(Portal)"]
            GITLAB["GitLab<br/>(SCM + CI/CD)"]
            VAULT["Vault<br/>(Secrets)"]
            KYVERNO["Kyverno<br/>(Policies)"]
        end
        
        subgraph INFRA["Infrastructure"]
            HARBOR["Harbor<br/>(Registry)"]
            KONG["Kong<br/>(Gateway)"]
            KAFKA["Kafka<br/>(Events)"]
            SONARQUBE["SonarQube<br/>(SAST)"]
        end
        
        subgraph OBSERVABILITY["Observability"]
            PROMETHEUS["Prometheus<br/>(Metrics)"]
            GRAFANA["Grafana<br/>(Dashboards)"]
            LOKI["Loki<br/>(Logs)"]
        end
        
        subgraph SECURITY["Security & TLS"]
            AUTHENTIK["Authentik<br/>(SSO/OIDC)"]
            CERTMGR["cert-manager<br/>(TLS)"]
            EXTSECRTS["External Secrets<br/>(Vault Integration)"]
        end
    end
    
    CP -->|manages| WORKERS
    ARGOCD -->|syncs| SERVICES
    ARGOCD -->|syncs| INFRA
    SERVICES -->|auth| AUTHENTIK
    SERVICES -->|secrets| VAULT
    SERVICES -->|policies| KYVERNO
    INFRA -->|monitored by| OBSERVABILITY
    SERVICES -->|metrics| PROMETHEUS
    CERTMGR -->|manages| KONG
    EXTSECRTS -->|pulls secrets| VAULT
    
    style CP fill:#fff9c4
    style WORKERS fill:#fff9c4
    style GITOPS fill:#c8e6c9
    style SERVICES fill:#bbdefb
    style INFRA fill:#ffe0b2
    style OBSERVABILITY fill:#f8bbd0
    style SECURITY fill:#e1bee7
```

---

## Level 3: Component (Key Container Internals)

### 3a. Backstage (IDP Portal)

```mermaid
graph TB
    subgraph CLIENT["Client Layer"]
        REACT["React Frontend<br/>- Catalog UI<br/>- K8s Viewer<br/>- ArgoCD Integration<br/>- Scorecard Plugin"]
    end
    
    subgraph APP["Application Layer"]
        API["Node.js Backend<br/>- Catalog API<br/>- OIDC/LDAP Auth<br/>- K8s Aggregator<br/>- Scaffolder API"]
    end
    
    subgraph DATA["Data Layer"]
        PG["PostgreSQL<br/>- Service Metadata<br/>- User Profiles<br/>- Plugin State"]
    end
    
    subgraph EXTERNAL["External Integrations"]
        AUTHZ["Authentik<br/>(OIDC Provider)"]
        K8SAPI["Kubernetes API<br/>(Metrics)"]
        ARGOAPI["ArgoCD API<br/>(Deployment Status)"]
    end
    
    REACT -->|REST API| API
    API -->|Query/Insert| PG
    API -->|OIDC| AUTHZ
    API -->|Query| K8SAPI
    API -->|Query| ARGOAPI
    REACT -->|OAuth 2.0| AUTHZ
    
    style REACT fill:#c8e6c9
    style API fill:#c8e6c9
    style PG fill:#ffccbc
    style AUTHZ fill:#e1bee7
    style K8SAPI fill:#bbdefb
    style ARGOAPI fill:#bbdefb
```

### 3b. DevSecOps Pipeline (git push → deployment)

```mermaid
graph LR
    DEV["Developer<br/>git push"] -->|triggers| GITLAB["GitLab CI"]
    
    GITLAB -->|Stage 1| SCAN["Security Scanning<br/>- Gitleaks<br/>- SonarQube<br/>- Trivy<br/>- Cosign Sign"]
    
    SCAN -->|Stage 2| BUILD["Build & Registry<br/>- Build OCI Image<br/>- Push to Harbor<br/>- Generate SBOM"]
    
    BUILD -->|Stage 3| VERIFY["Verify & Comply<br/>- Image Signature<br/>- Supply Chain Checks<br/>- Audit Log"]
    
    VERIFY -->|Stage 4| GITOPS["GitOps Sync<br/>- Update Manifests<br/>- Push to Git"]
    
    GITOPS -->|triggers| ARGOCD["ArgoCD Watches<br/>- Detects Changes<br/>- Applies to K8s"]
    
    ARGOCD -->|validates| KYVERNO["Kyverno Policies<br/>- Image Registry OK?<br/>- Resource Quotas?<br/>- Network Policies?"]
    
    KYVERNO -->|deploys| POD["Pod Running<br/>- Prometheus Scrape<br/>- Grafana Monitor<br/>- Kong Route"]
    
    style DEV fill:#e3f2fd
    style GITLAB fill:#fff9c4
    style SCAN fill:#ffe0b2
    style BUILD fill:#ffe0b2
    style VERIFY fill:#f8bbd0
    style GITOPS fill:#c8e6c9
    style ARGOCD fill:#c8e6c9
    style KYVERNO fill:#c8e6c9
    style POD fill:#bbdefb
```

---

## Level 4: Code (Specifics)

Key directories in this repository:

```mermaid
graph TB
    subgraph REPO["homelab-idp/"]
        README["README.md<br/>(Overview + Architecture)"]
        DOCS["docs/<br/>- getting-started.md<br/>- C4-ARCHITECTURE.md"]
        MANIFESTS["manifests/<br/>- backstage/<br/>- kafka/<br/>- kyverno/<br/>- vault/<br/>- sonarqube/<br/>- argocd/<br/>- harbor/<br/>- kong/<br/>- authentik/<br/>- monitoring/"]
        ENV["config/<br/>- talos-machine-config.yaml<br/>- network-config.yaml<br/>.env.example"]
    end
    
    README -->|describes| MANIFESTS
    DOCS -->|guides deployment of| MANIFESTS
    ENV -->|configures| MANIFESTS
    
    style README fill:#c8e6c9
    style DOCS fill:#c8e6c9
    style MANIFESTS fill:#bbdefb
    style ENV fill:#ffe0b2
```

---

## Data Flow Summary

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant GIT as GitLab
    participant CI as GitLab CI
    participant HARBOR as Harbor
    participant ARGOCD as ArgoCD
    participant K8S as Kubernetes
    participant MON as Prometheus
    
    DEV->>GIT: git push feature/new-service
    GIT->>CI: Trigger pipeline
    CI->>CI: Run security scans (Gitleaks, SAST)
    CI->>HARBOR: Build & push OCI image
    CI->>GIT: Update deployment manifests
    ARGOCD->>GIT: Watch for changes
    ARGOCD->>K8S: Apply manifests
    K8S->>K8S: Kyverno validates policies
    K8S->>K8S: Pod scheduled & started
    K8S->>MON: Export metrics
    MON->>MON: Scrape metrics every 30s
```

---

## Security Boundaries

```mermaid
graph TB
    subgraph UNTRUSTED["UNTRUSTED (Internet)"]
        USERS["External Users"]
    end
    
    subgraph GATEWAY["SEMI-TRUSTED (Boundary)"]
        CF["Cloudflare Tunnel<br/>- DDoS Protection<br/>- TLS Termination"]
        KONG["Kong Gateway<br/>- Rate Limiting<br/>- OIDC Enforcement"]
    end
    
    subgraph CLUSTER["TRUSTED (Inside Cluster)"]
        AUTH["Authentik<br/>(OIDC Provider)"]
        K8S["Kubernetes Cluster<br/>- mTLS Service-to-Service<br/>- RBAC Enforcement<br/>- Network Policies<br/>- Kyverno Admission"]
        VAULT["Vault<br/>(Secret Management)"]
    end
    
    USERS -->|HTTPS| CF
    CF -->|HTTPS| KONG
    KONG -->|OAuth 2.0| AUTH
    KONG -->|Allowed| K8S
    K8S -->|Retrieve Secrets| VAULT
    
    style UNTRUSTED fill:#ffebee
    style GATEWAY fill:#fff3e0
    style CLUSTER fill:#e8f5e9
```

---

## Key Design Decisions (ADRs)

### ADR #001: Talos Linux as OS
- **Decision:** Immutable, API-driven OS — no shell, no SSH
- **Rationale:** Impossible to misconfigure at runtime; forces GitOps discipline
- **Trade-off:** Steep learning curve vs. operational safety

### ADR #002: ArgoCD + Git as Source of Truth
- **Decision:** All infrastructure declared in Git, synced by ArgoCD
- **Rationale:** Audit trail, rollback capability, disaster recovery
- **Trade-off:** No kubectl apply from laptop; enforces best practices

### ADR #003: Vault + External Secrets Operator
- **Decision:** Dynamic secrets + K8s lifecycle separation
- **Rationale:** Secret rotation, temporary credentials, better than Sealed Secrets
- **Trade-off:** More complex setup, but production-grade

### ADR #004: Kyverno for Policy Enforcement
- **Decision:** Native Kubernetes CRDs instead of OPA/Gatekeeper
- **Rationale:** Easier to write/audit than Rego; lower cognitive load
- **Trade-off:** Less powerful than OPA, but 80/20 for most cases

---

## What I Learned

✅ **Golden Path Works** — Once defined, deployment time: 2 weeks → 5 minutes

✅ **Immutable Infrastructure >> Config Management** — Talos forced better practices

✅ **GitOps is Non-Negotiable** — Can recover from any disaster in <5 minutes

✅ **Monitoring ≠ Observability** — Prometheus shows "what broke", tracing shows "why"

✅ **Security by Default** — Kyverno policies caught 80% of misconfigurations before they existed

⚠️ **Self-Hosted ≠ Fun** — GitLab requires babysitting; managed GitHub would reduce ops by ~30%

⚠️ **Kafka is Overkill for Learning** — Event streaming is powerful, but Redis streams are simpler first

---

## Next Steps

- For deployment: see [docs/getting-started.md](./getting-started.md)
- For infrastructure code: see [../manifests/](../manifests/)
- For GitOps workflow: see [docs/gitops-workflow.md](./gitops-workflow.md) (coming soon)
