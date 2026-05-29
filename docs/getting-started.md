# Getting Started

## Prerequisites

- 3+ bare-metal nodes (or VMs) for control-plane, 3+ for workers
- `talosctl` installed locally
- `kubectl` installed locally
- `helm` v3 installed locally
- `argocd` CLI installed locally
- Cloudflare account with a domain
- Domain DNS managed by Cloudflare

## 1. Provision Talos

Download the Talos factory image and boot each node from ISO.
See [Talos docs](https://www.talos.dev/latest/introduction/getting-started/).

```bash
# Apply control-plane config
talosctl apply-config --insecure --nodes <NODE_IP> --file controlplane.yaml

# Apply worker config
talosctl apply-config --insecure --nodes <NODE_IP> --file worker.yaml

# Bootstrap etcd (only once, on first control-plane node)
talosctl bootstrap --nodes <FIRST_CP_IP>

# Get kubeconfig
talosctl kubeconfig --nodes <FIRST_CP_IP>
```

## 2. Install MetalLB

```bash
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb -n metallb-system --create-namespace
kubectl apply -f manifests/metallb-pool.yaml
```

## 3. Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set installCRDs=true

# Create Cloudflare API token secret
kubectl create secret generic cloudflare-api-token \
  -n cert-manager \
  --from-literal=api-token=<CLOUDFLARE_API_TOKEN>

kubectl apply -f manifests/cert-manager-issuers.yaml
```

## 4. Install Kong

```bash
helm repo add kong https://charts.konghq.com
helm install kong kong/ingress -n kong --create-namespace \
  -f manifests/kong/values.yaml
```

## 5. Install Vault

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault -n vault --create-namespace \
  -f manifests/vault/values.yaml

# Initialize Vault (save the unseal keys securely!)
kubectl exec -n vault vault-0 -- vault operator init

# Unseal
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY_1>
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY_2>
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY_3>
```

## 6. Install External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace

kubectl apply -f manifests/external-secrets/
```

## 7. Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace \
  -f manifests/argocd/values.yaml
```

Once ArgoCD is running, add the remaining services as ArgoCD Applications
using the `argocd-app.yaml` files in each service directory.

## 8. Deploy remaining services via ArgoCD

```bash
# Each service has an ArgoCD Application manifest
kubectl apply -f manifests/gitlab/argocd-app.yaml     # after configuring values
kubectl apply -f manifests/harbor/argocd-app.yaml
kubectl apply -f manifests/authentik/argocd-app.yaml
kubectl apply -f manifests/sonarqube/argocd-app.yaml
kubectl apply -f manifests/monitoring/argocd-app.yaml
kubectl apply -f manifests/backstage/04-argocd-app.yaml
kubectl apply -f manifests/kafka/argocd-app.yaml
kubectl apply -f manifests/kyverno/argocd-app.yaml
```

## 9. Configure Cloudflare Tunnel

```bash
# Create tunnel
cloudflared tunnel create homelab

# Get tunnel credentials and create K8s secret
cloudflared tunnel token --cred-file credentials.json <TUNNEL_ID>
kubectl create secret generic cloudflared-credentials \
  -n cloudflared \
  --from-file=credentials.json

kubectl apply -f manifests/cloudflared/deployment.yaml
```
