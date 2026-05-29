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

## Getting Started

See [`docs/getting-started.md`](docs/getting-started.md) for full setup guide.

Requirements:
- 6 nodes with Talos Linux installed
- `talosctl` and `kubectl` configured
- Cloudflare account + API token
- Domain configured in Cloudflare DNS

---

## Key Design Decisions

**Why Talos instead of k3s/kubeadm?**  
Immutable OS — no shell, no SSH, API-only. Harder to set up once, impossible to misconfigure at runtime. Forces proper GitOps discipline.

**Why self-hosted GitLab instead of GitHub?**  
Full control over CI runners, registry, and secrets. GitLab's built-in registry + CI removes one integration point. In production, GitHub would work fine.

**Why Vault + External Secrets instead of Sealed Secrets?**  
Vault enables dynamic secrets and secret rotation. External Secrets Operator keeps the K8s secret lifecycle separate from the app lifecycle. Sealed Secrets would work for a simpler setup.

**Why Kyverno instead of OPA/Gatekeeper?**  
Native K8s resources (ClusterPolicy CRDs) — easier to read, write, and audit. Gatekeeper requires Rego, which adds cognitive overhead for a team already dealing with YAML.

---

## Author

**Gerardo Lucero** — DevSecOps Engineer  
[LinkedIn](https://linkedin.com/in/luceroriosg) · [GitHub](https://github.com/GerardoLucero)
