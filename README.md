# Kubernetes Configs

GitOps repository for a self-hosted Kubernetes cluster running on Talos Linux, using Cilium for networking and ArgoCD for continuous deployment.

## Overview

This repository contains the configuration for a single-node (or multi-node) Kubernetes cluster that automatically deploys and manages applications through ArgoCD's ApplicationSet pattern. Once bootstrapped, ArgoCD watches this repository and automatically syncs any changes.

**Key Components:**
- **Talos Linux** - Immutable, secure Kubernetes OS
- **Cilium** - CNI with Gateway API, L2 announcements, and kube-proxy replacement
- **ArgoCD** - GitOps continuous deployment
- **Longhorn** - Distributed block storage
- **External Secrets + 1Password** - Secrets management
- **CloudNativePG** - PostgreSQL operator for stateful workloads

## Prerequisites

Install the following tools before proceeding:

| Tool | Purpose | Installation |
|------|---------|--------------|
| `talosctl` | Talos cluster management | [talos.dev/docs](https://www.talos.dev/latest/introduction/getting-started/) |
| `kubectl` | Kubernetes CLI | [kubernetes.io/docs](https://kubernetes.io/docs/tasks/tools/) |
| `helm` | Kubernetes package manager | [helm.sh/docs](https://helm.sh/docs/intro/install/) |
| `yq` | YAML processor | [github.com/mikefarah/yq](https://github.com/mikefarah/yq#install) |

## Repository Structure

```
funland/                          # Cluster name
├── argocd/argocd/               # ArgoCD + ApplicationSet
├── kube-system/cilium/          # Cilium CNI configuration
├── gateway/gateway/             # Gateway API resources (internal/external)
├── gateway-api/gateway-api-crds/ # Gateway API CRD definitions
├── cert-manager/cert-manager/   # TLS certificate management
├── certificate/certificate/     # Wildcard certificates
├── external-secrets/            # External Secrets Operator + 1Password
├── cnpg/                        # CloudNativePG operator + database clusters
├── longhorn-system/longhorn/    # Persistent storage
├── monitoring/                  # Prometheus, Grafana, Loki, Promtail, metrics-server
└── [app-namespace]/[app]/       # Application deployments
```

Each application follows the pattern `namespace/app-name/` containing:
- `application.yaml` - ArgoCD Application (for Helm charts)
- `http-route.yaml` - Gateway API routing
- `deployment.yaml` / `kustomization.yaml` - Raw manifests or Kustomize

## Cluster Bootstrap

### 1. Talos Setup

Create a `patch.yaml` to disable the default CNI and kube-proxy (Cilium will replace both):

```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

Set environment variables for your cluster:

```sh
export TALOS_IP="192.168.10.30"      # Control plane IP
export TALOS_API_PORT="6443"         # Kubernetes API port
export CLUSTER_NAME="funland"        # Cluster name
```

Generate configs and bootstrap the cluster:

```sh
# Generate Talos configuration
talosctl gen config "${CLUSTER_NAME}" "https://${TALOS_IP}:${TALOS_API_PORT}" --config-patch @patch.yaml

# Apply configuration to the node (this starts the bootstrap)
talosctl apply-config --insecure -n "${TALOS_IP}" --file controlplane.yaml

# Bootstrap the cluster (wait for node to be reachable first)
talosctl bootstrap --nodes "${TALOS_IP}" --endpoints "${TALOS_IP}" --talosconfig=./talosconfig

# Get kubeconfig for kubectl access
talosctl kubeconfig --nodes "${TALOS_IP}" --endpoints "${TALOS_IP}" --talosconfig=./talosconfig
```

> **Note:** The node will show as `NotReady` until Cilium is installed. This is expected since no CNI is configured.

### 2. Install Gateway API CRDs

Install the Gateway API CRDs before Cilium (Cilium's Gateway API integration requires these):

```sh
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
```

### 3. Install Cilium

Create the certificate namespace (used by Cilium Gateway API for TLS secrets):

```sh
kubectl create namespace certificate
```

Install Cilium using the configuration from this repository:

```sh
export cilium_applicationyaml=$(curl -sL "https://raw.githubusercontent.com/mpearlst/kubernetes-configs/refs/heads/main/funland/kube-system/cilium/application.yaml" | yq eval-all '. | select(.metadata.name == "cilium-application" and .kind == "Application")' -)
export cilium_name=$(echo "$cilium_applicationyaml" | yq eval '.metadata.name' -)
export cilium_chart=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.chart' -)
export cilium_repo=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.repoURL' -)
export cilium_namespace=$(echo "$cilium_applicationyaml" | yq eval '.spec.destination.namespace' -)
export cilium_version=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.targetRevision' -)
export cilium_values=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.helm.valuesObject' -)

echo "$cilium_values" | helm template $cilium_name $cilium_chart \
  --repo $cilium_repo \
  --version $cilium_version \
  --namespace $cilium_namespace \
  --values - | kubectl apply --namespace $cilium_namespace --filename -
```

**Verify Cilium is running:**

```sh
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium
kubectl get nodes  # Should now show Ready
```

### 4. Install ArgoCD

Install ArgoCD and the ApplicationSet that will manage all other applications:

```sh
export argocd_applicationyaml=$(curl -sL "https://raw.githubusercontent.com/mpearlst/kubernetes-configs/refs/heads/main/funland/argocd/argocd/application.yaml" | yq eval-all '. | select(.metadata.name == "argocd-application" and .kind == "Application")' -)
export argocd_name=$(echo "$argocd_applicationyaml" | yq eval '.metadata.name' -)
export argocd_chart=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.chart' -)
export argocd_repo=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.repoURL' -)
export argocd_namespace=$(echo "$argocd_applicationyaml" | yq eval '.spec.destination.namespace' -)
export argocd_version=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.targetRevision' -)
export argocd_values=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.helm.valuesObject' - | yq eval 'del(.configs.cm)' -)
export argocd_config=$(curl -sL "https://raw.githubusercontent.com/mpearlst/kubernetes-configs/refs/heads/main/funland/argocd/argocd/appset.yaml" | yq eval-all '. | select(.kind == "AppProject" or .kind == "ApplicationSet")' -)

# Install ArgoCD
echo "$argocd_values" | helm template $argocd_name $argocd_chart \
  --repo $argocd_repo \
  --version $argocd_version \
  --namespace $argocd_namespace \
  --values - | kubectl apply --namespace $argocd_namespace --filename -

# Apply ApplicationSet (this triggers automatic deployment of all apps)
echo "$argocd_config" | kubectl apply --filename -
```

**Verify ArgoCD is running:**

```sh
kubectl get pods -n argocd
```

## Post-Setup

### Accessing ArgoCD Dashboard

Get the initial admin password:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Port-forward to access the UI:

```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open https://localhost:8080 (username: `admin`)

### Verify Applications

Once ArgoCD is running, it will automatically discover and deploy all applications from this repository. Check status:

```sh
# List all ArgoCD applications
kubectl get applications -n argocd

# Check for any sync issues
kubectl get applications -n argocd -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.sync.status}{"\t"}{.status.health.status}{"\n"}{end}'
```

## Deployed Applications

| Application | Namespace | Description |
|-------------|-----------|-------------|
| AdGuard Home | adguard | DNS ad-blocking |
| Authentik | authentik | Identity provider / SSO |
| Cert-Manager | cert-manager | TLS certificate automation |
| Cilium | kube-system | CNI with Gateway API, Hubble observability |
| Grafana | monitoring | Metrics visualization |
| Hubble | kube-system | Network flow observability UI |
| Immich | immich | Photo/video management |
| Jellyfin | media | Media server |
| Loki | monitoring | Log aggregation |
| Longhorn | longhorn-system | Distributed storage |
| n8n | n8n | Workflow automation |
| Newt | newt | Pangolin tunnel client |
| ntfy | ntfy | Push notifications |
| Pocket ID | pocket-id | OIDC provider |
| Privatebin | privatebin | Encrypted pastebin |
| Prometheus | monitoring | Metrics collection |
| Promtail | monitoring | Log shipping agent |
| Synapse | synapse | Matrix homeserver |
| Vaultwarden | vaultwarden | Bitwarden-compatible password manager |
| Vikunja | vikunja | Task management |

## Network Architecture

- **IP Pool:** `192.168.10.30` - `192.168.10.49` (Cilium L2 announcements)
- **Internal Gateway:** `192.168.10.30` - Routes internal services via HTTPRoute resources
- **External Gateway:** `192.168.10.31` - Routes public-facing services
- **CNI:** Cilium with kube-proxy replacement and Gateway API enabled

## Observability Stack

### Metrics (Prometheus + Grafana)
- Prometheus scrapes metrics from all services with ServiceMonitors
- Cilium exports network metrics (DNS, TCP, HTTP, drops, flows)
- Grafana dashboards available at `grafana.batlab.io`

### Logging (Loki + Promtail)
- Promtail runs as a DaemonSet, collecting logs from all pods
- Loki aggregates logs in SingleBinary mode with Longhorn storage
- Query logs via Grafana's Explore tab

### Network Observability (Hubble)
- Hubble UI available at `hubble.batlab.io`
- Real-time network flow visualization
- DNS query tracking and HTTP request inspection

## Database Backups (CloudNativePG)

All PostgreSQL clusters are configured with automated backups to Backblaze B2:

| Database | Backup Path |
|----------|-------------|
| authentik-dbc | `s3://batcave-kubernetes/funland/postgres-backups/authentik` |
| immich-dbc | `s3://batcave-kubernetes/funland/postgres-backups/immich` |
| n8n-dbc | `s3://batcave-kubernetes/funland/postgres-backups/n8n` |
| synapse-dbc | `s3://batcave-kubernetes/funland/postgres-backups/synapse` |
| vaultwarden-dbc | `s3://batcave-kubernetes/funland/postgres-backups/vaultwarden` |
| vikunja-dbc | `s3://batcave-kubernetes/funland/postgres-backups/vikunja` |

**Backup Configuration:**
- **Retention:** 30 days
- **Compression:** Snappy
- **Encryption:** AES256 (WAL and data)
- **Schedule:** Daily at midnight (via ScheduledBackup resources)
- **Continuous:** WAL archiving enabled for point-in-time recovery

**Verify backups:**
```sh
kubectl get scheduledbackups -n postgres
kubectl get backups -n postgres
```

## Troubleshooting

### Node stuck in NotReady
Cilium is not running. Check Cilium pods: `kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium`

### ArgoCD apps not syncing
1. Check ApplicationSet: `kubectl get applicationset -n argocd`
2. View app status: `kubectl describe application <app-name> -n argocd`
3. Check ArgoCD logs: `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller`

### Services not accessible
1. Verify Gateway: `kubectl get gateway -A`
2. Check HTTPRoutes: `kubectl get httproute -A`
3. Verify L2 announcements: `kubectl get ciliuml2announcementpolicy,ciliumloadbalancerippool`

### Storage issues
Check Longhorn dashboard or pods: `kubectl get pods -n longhorn-system`

## External Documentation

- [Talos Linux](https://www.talos.dev/latest/)
- [Cilium](https://docs.cilium.io/en/stable/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/)
- [Longhorn](https://longhorn.io/docs/)
- [CloudNativePG](https://cloudnative-pg.io/documentation/)
