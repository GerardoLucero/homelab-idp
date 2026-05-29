# Stack Decisions

## Why each tool was chosen

### Talos Linux (Kubernetes OS)
Immutable, API-driven OS. No SSH, no shell access — every change goes through the API.
Forces proper GitOps: if you can't do it via API or YAML, you don't do it.
Harder initial setup, dramatically more stable and secure at runtime.

**Alternative considered:** k3s — lighter, easier, but allows shell access which creates drift risk.

### ArgoCD (GitOps)
Every deployment is a Git commit. No `kubectl apply` in CI — the pipeline pushes to the infra repo and ArgoCD syncs.
Pull-based model: the cluster pulls desired state from Git, not the CI runner pushing to the cluster.

### GitLab (SCM + CI)
Full control over runners, registry, and secrets. Built-in container registry removes one integration.
CI/CD pipeline: Gitleaks → SonarQube → Trivy → Cosign → gitops commit → ArgoCD sync.

### Harbor (Container Registry)
Enterprise-grade registry with:
- Cosign integration for image signing (supply chain security)
- Trivy scanning built-in (scan on push)
- SBOM generation per image
- Project-level RBAC

**Alternative considered:** GHCR — simpler, but no built-in scanning or signing UI.

### Backstage (IDP)
Unified developer portal: software catalog, Kubernetes metrics, ArgoCD app status, SonarQube scores — all in one place.
Registered all homelab services with `kubernetes-label-selector` for live K8s metrics.

### Vault + External Secrets (Secrets)
Zero plaintext secrets in any manifest or Git repo.
ESO (External Secrets Operator) syncs Vault secrets → K8s Secrets automatically.
Supports secret rotation without redeploying apps.

### Authentik (Identity)
SSO/OIDC for all services: ArgoCD, Grafana, GitLab, Backstage.
Single login → all tools. SAML support for legacy integrations.

### Kyverno (Policy)
Admission webhook policies enforced at cluster level:
- No `:latest` image tags
- Required resource limits on all containers
- No privileged containers
- runAsNonRoot required
- Required `app` and `version` labels

These are the same policies that block things like the GitLab CI runner pods.

### Kong (API Gateway)
Ingress + API management. ForwardAuth to Authentik for single-sign-on at the gateway level.
Rate limiting, auth plugins, and routing all in one place.

### Kafka + Strimzi (Streaming)
Production-grade Kafka via Strimzi operator.
Demo: Debezium CDC pipeline → PostgreSQL changes → Kafka topics → Kafka Connect.

### Longhorn (Storage)
Distributed block storage across all worker nodes.
Replicated volumes survive node failures.
Cheaper and simpler than Ceph for a homelab scale.

### cert-manager + Cloudflare (TLS)
Automatic Let's Encrypt certificates via DNS-01 challenge (Cloudflare).
All services get valid HTTPS — no self-signed certificates anywhere.

### Cloudflare Tunnel (External Access)
Zero open ports on the home router. Cloudflare Tunnel creates an outbound-only connection.
No port forwarding, no exposed IP, no DDoS surface. Free tier covers this use case.
