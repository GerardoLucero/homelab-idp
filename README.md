# homelab-idp

A production-grade **Internal Developer Platform (IDP)** running on a 6-node Kubernetes cluster at home.

This repo contains the infrastructure manifests, Helm values templates, and architecture docs for a full Platform Engineering stack — built to practice real-world skills, not sandbox demos.

---

## Stack

| Layer | Tool | Purpose |
|---|---|---|
| **Kubernetes** | Talos Linux v1.13 | Immutable, API-driven OS — no SSH |
| **GitOps** | ArgoCD | All deployments as code |
| **Source Control** | GitLab | Self-hosted SCM + CI/CD |
| **Registry** | Harbor | Container images + Cosign signing + SBOM |
| **IDP** | Backstage | Internal Developer Portal — catalog, K8s metrics, ArgoCD, SonarQube |
| **Secret Management** | Vault + External Secrets | Zero plaintext secrets in manifests |
| **Identity** | Authentik | SSO/OIDC for all services |
| **API Gateway** | Kong | Ingress + rate limiting + auth |
| **Policy Engine** | Kyverno | Admission control — enforce security baseline |
| **SAST** | SonarQube | Code quality + security scanning |
| **Streaming** | Apache Kafka (Strimzi) | Event streaming + Debezium CDC demo |
| **Monitoring** | Prometheus + Grafana | Metrics, dashboards, alerts |
| **Storage** | Longhorn | Distributed block storage |
| **TLS** | cert-manager + Let's Encrypt | Automatic certificate management |
| **Tunnel** | Cloudflare Tunnel | Zero-trust external access — no open ports |

---

## Architecture

```
Internet
   │
   └── Cloudflare Tunnel (zero-trust, no open ports)
          │
          └── Kong API Gateway (ingress + auth)
                 │
                 ├── Authentik (SSO/OIDC)
                 │
                 ├── Traefik (internal routing)
                 │     ├── Backstage    :7007
                 │     ├── GitLab       :80/443
                 │     ├── ArgoCD       :443
                 │     ├── Harbor       :443
                 │     ├── SonarQube    :9000
                 │     ├── Grafana      :3000
                 │     ├── Vault        :8200
                 │     └── Kafka UI     :8080
                 │
                 └── Kubernetes Cluster (Talos v1.13)
                        ├── control-plane × 3  (192.168.68.11-13)
                        └── worker       × 3  (192.168.68.21-23)
```

---

## Hardware

6× **HP EliteDesk 800 G4 Mini**
- CPU: Intel Core i5-8500T (6 cores)
- RAM: 16GB DDR4
- Storage: 256GB NVMe SSD + 1TB HDD (Longhorn)
- Network: 1GbE

---

## DevSecOps Pipeline

Every service deployed through this cluster goes through:

```
git push
  → GitLab CI
      ├── Gitleaks       (secret scanning)
      ├── SonarQube      (SAST + quality gate)
      ├── Trivy          (container CVE scan)
      ├── Cosign sign    (image signing)
      └── ArgoCD sync    (GitOps deploy)
              └── Kyverno (policy enforcement at admission)
```

---

## Repository Structure

```
homelab-idp/
├── architecture/          # Diagrams and stack decisions
├── manifests/
│   ├── backstage/         # IDP deployment + RBAC
│   ├── kafka/             # Strimzi cluster + connectors + CDC demo
│   ├── kyverno/policies/  # Security baseline policies
│   ├── vault/             # Secret management
│   ├── sonarqube/         # SAST platform
│   ├── external-secrets/  # ESO + Vault integration
│   ├── argocd/            # GitOps engine values
│   ├── harbor/            # Container registry values
│   ├── kong/              # API gateway values
│   ├── authentik/         # Identity provider values
│   └── monitoring/        # Prometheus + Grafana values
├── docs/                  # Installation guides
└── .env.example           # Required variables reference
```

All secrets replaced with `<REPLACE_ME>` or `<BASE64_ENCODED_VALUE>` placeholders.  
Use Vault + External Secrets Operator to manage real values.

---

## Quick Start (5 minutes verification)

If you already have the cluster running, verify the stack:

```bash
# 1. Check cluster health
kubectl get nodes -o wide
# Expected: 6 nodes (3 control-plane, 3 worker) in Ready state

# 2. Verify core services are running
kubectl get pods -n argocd
kubectl get pods -n backstage
kubectl get pods -n sonarqube

# 3. Access Backstage (IDP portal)
# Forward to localhost:7007
kubectl port-forward -n backstage svc/backstage 7007:7007
# Open: http://localhost:7007
# Default credentials in Vault (see docs/getting-started.md)

# 4. Verify GitOps sync
argocd app list
# Expected: All apps in Synced state
```

**Full setup guide:** See [`docs/getting-started.md`](docs/getting-started.md)

**Prerequisites for new deployment:**
- 6 nodes with Talos Linux v1.13+ installed
- `talosctl` and `kubectl` configured
- Cloudflare account + API token
- Domain configured in Cloudflare DNS

---

## Key Design Decisions (Architecture Decision Records)

### 1. Talos Linux as OS (ADR #001)
**Why Talos instead of k3s/kubeadm?**
- **Decision:** Immutable, API-driven OS — no shell, no SSH access
- **Rationale:** Harder to set up once, impossible to misconfigure at runtime. Forces GitOps discipline from day one.
- **Trade-off:** Steep learning curve vs. operational safety and consistency
- **Status:** ✅ Validated in production-like environment

### 2. GitOps-First with ArgoCD (ADR #002)
**Why ArgoCD + Git as source of truth?**
- **Decision:** All infrastructure and apps declared in Git, synced by ArgoCD
- **Rationale:** Audit trail, rollback capability, disaster recovery, and team collaboration
- **Trade-off:** Requires discipline (no kubectl apply from laptop), but enforces best practices
- **Status:** ✅ Used for 100% of workload deployments

### 3. Vault + External Secrets Operator (ADR #003)
**Why Vault instead of Sealed Secrets?**
- **Decision:** Vault (dynamic secrets) + ESO (K8s lifecycle separation)
- **Rationale:** 
  - Vault enables secret rotation and dynamic secrets (e.g., temporary DB credentials)
  - ESO keeps secret lifecycle separate from application lifecycle
  - Better for teams with security-focused operations
- **Trade-off:** More complex than Sealed Secrets, but production-grade
- **Status:** ✅ All 50+ secrets managed via Vault

### 4. Kyverno for Policy Enforcement (ADR #004)
**Why Kyverno instead of OPA/Gatekeeper?**
- **Decision:** Native Kubernetes resources (ClusterPolicy CRDs)
- **Rationale:** Easier to write/audit than Rego; lower cognitive load for YAML-familiar teams
- **Trade-off:** Less powerful than OPA, but 80/20 for most use cases
- **Status:** ✅ Enforcing: image registry whitelist, resource quotas, network policies

### 5. Self-Hosted GitLab vs. GitHub
**Why self-hosted GitLab?**
- **Decision:** Full control over CI runners, registry, secrets management
- **Rationale:** Built-in registry + CI removes integration points; better for learning GitOps
- **Trade-off:** More operational overhead than GitHub. In production, GitHub would be simpler.
- **Status:** ⚠️ Works, but requires regular maintenance

---

## What I Learned (Architecture Lessons)

✅ **Golden Path Works** — Once Golden Path was defined (repo template + helm charts + pipeline), deploying new services dropped from 2 weeks to 5 minutes

✅ **Immutable Infrastructure >> Configuration Management** — Talos's API-driven approach forced better practices than traditional VMs with SSH

✅ **GitOps is Non-Negotiable** — Every deployment through Git + ArgoCD meant we could recover from any disaster in <5 minutes

✅ **Monitoring ≠ Observability** — Prometheus alone shows "what broke", but only distributed tracing (Jaeger) shows "why it broke"

✅ **Security by Default > Security Tooling** — Kyverno policies enforced at admission time prevented 80% of misconfigurations before they existed

⚠️ **Self-Hosted ≠ Fun** — GitLab is powerful but requires babysitting; managed GitHub would reduce ops burden by ~30%

⚠️ **Kafka is Overkill for Learning** — Event streaming is powerful, but for a homelab, would use Redis streams or PostgreSQL events first

---

## Author

**Gerardo Lucero** — DevSecOps Engineer  
[LinkedIn](https://linkedin.com/in/luceroriosg) · [GitHub](https://github.com/GerardoLucero)
