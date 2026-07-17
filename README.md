# Kubernetes Configs

GitOps repository for a self-hosted Kubernetes cluster running on Talos Linux, using Cilium for networking and ArgoCD for continuous deployment.

## Overview

This repository contains the configuration for a single-node (or multi-node) Kubernetes cluster that automatically deploys and manages applications through ArgoCD's ApplicationSet pattern. Once bootstrapped, ArgoCD watches this repository and automatically syncs any changes.

**Key Components:**
- **Talos Linux** - Immutable, secure Kubernetes OS
- **Cilium** - CNI with Gateway API, BGP peering, and kube-proxy replacement
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
├── monitoring/                  # Prometheus, Grafana, metrics-server
├── talos/                       # Talos/Kubernetes version tracking (talhelper)
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
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
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
| ArgoCD | argocd | GitOps continuous deployment |
| Authentik | authentik | Identity provider / SSO |
| Cert-Manager | cert-manager | TLS certificate automation |
| Certificate | certificate | Wildcard TLS certificate |
| Cilium | kube-system | CNI with Gateway API, BGP, Hubble observability |
| CloudNativePG | postgres / cnpg-system | PostgreSQL operator + database clusters |
| Cloudflare Tunnel | cloudflare | Cloudflare tunnel ingress controller |
| External Secrets Operator | external-secrets | Syncs secrets from 1Password |
| Gateway | gateway | Gateway API Gateway resources (internal/external) |
| Gateway API CRDs | gateway-api | Gateway API CRD definitions |
| Grafana | monitoring | Metrics visualization |
| Immich | immich | Photo/video management |
| Jellyfin | media | Media server |
| Longhorn | longhorn-system | Distributed storage |
| metrics-server | kube-system | Kubernetes resource metrics API |
| ntfy | ntfy | Push notifications |
| OpenSpeedTest | openspeedtest | Network speed testing |
| 1Password Connect | 1password-connect | 1Password Connect server (ESO backend) |
| Privatebin | privatebin | Encrypted pastebin |
| Prometheus | monitoring | Metrics collection (kube-prometheus-stack) |
| Vaultwarden | vaultwarden | Bitwarden-compatible password manager |

## Network Architecture

- **IP Pool:** `192.168.101.0/24` (Cilium BGP advertisements, peered with UniFi router at ASN 64514)
- **Internal Gateway:** `192.168.101.1` - Routes internal services via HTTPRoute resources
- **External Gateway:** `192.168.101.2` - Routes public-facing services
- **CNI:** Cilium with kube-proxy replacement, BGP control plane, and Gateway API enabled

## Observability Stack

### Metrics (Prometheus + Grafana)
- Prometheus scrapes metrics from all services with ServiceMonitors
- Cilium exports network metrics (DNS, TCP, HTTP, drops, flows)
- Grafana dashboards available at `grafana.batlab.io`

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
| vaultwarden-dbc | `s3://batcave-kubernetes/funland/postgres-backups/vaultwarden` |

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

## Upgrading Talos & Kubernetes

`funland/talos/talconfig.yaml` ([talhelper](https://github.com/budimanjojo/talhelper) config) is the source of truth for the Talos OS version (`talosVersion`) and the Kubernetes version (`kubernetesVersion`). It also declares the Longhorn-required Talos system extensions (`siderolabs/iscsi-tools`, `siderolabs/util-linux-tools`) applied to every node.

Renovate watches this file (see `.github/renovate.json`) and opens a PR whenever a new Talos or Kubernetes release is published. These PRs are **never auto-merged** — a node OS / control-plane bump needs human review and a controlled rollout, unlike the chart/image bumps elsewhere in this repo.

Argo CD is not involved in this process: it only syncs Kubernetes API resources, while Talos/Kubernetes core upgrades go through the separate Talos API (`talosctl`). Merging the Renovate PR just records the target version — applying it is a manual runbook:

```sh
# Regenerate machine configs from the updated talconfig.yaml.
# This also resolves the current schematic (Longhorn extensions) into an
# installer image reference for each node, written to clusterconfig/*.yaml.
talhelper genconfig

# Look up the resolved installer image for a node, e.g.:
yq '.machine.install.image' clusterconfig/funland-funland-worker-1.yaml
# -> factory.talos.dev/installer/<schematic-id>:<new-talos-version>

# Roll the Talos OS upgrade node-by-node - workers first, control plane last.
# Validate cluster/Longhorn health between each node before continuing.
talosctl upgrade --nodes 192.168.10.21 --image <installer-image-from-above>
talosctl upgrade --nodes 192.168.10.22 --image <installer-image-from-above>
talosctl upgrade --nodes 192.168.10.23 --image <installer-image-from-above>
talosctl upgrade --nodes 192.168.10.30 --image <installer-image-from-above>

# Once every node is on the new Talos version, bump Kubernetes components
talosctl upgrade-k8s --nodes 192.168.10.30 --to <new-kubernetes-version>
```

> **Note:** `talhelper` also needs a `talsecret.yaml` (cluster/node secrets) to generate machine configs. Generate it locally with `talhelper gensecret` — it's gitignored and must never be committed in plaintext.

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
3. Verify BGP peering: `kubectl get ciliumbgpclusterconfig,ciliumbgppeerconfig,ciliumloadbalancerippool`

### Storage issues
Check Longhorn dashboard or pods: `kubectl get pods -n longhorn-system`

## External Documentation

- [Talos Linux](https://www.talos.dev/latest/)
- [Cilium](https://docs.cilium.io/en/stable/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/)
- [Longhorn](https://longhorn.io/docs/)
- [CloudNativePG](https://cloudnative-pg.io/documentation/)
