# Kubernetes Configs

GitOps repository for a self-hosted Kubernetes cluster running on Talos Linux, using Cilium for networking and ArgoCD for continuous deployment.

## Overview

This repository contains the configuration for the `funland` cluster (one control-plane node and three workers, defined in `funland/talos/talconfig.yaml`), which automatically deploys and manages applications through ArgoCD's ApplicationSet pattern. Once bootstrapped, ArgoCD watches this repository and automatically syncs any changes.

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
| `talhelper` | Generates Talos machine configs from `talconfig.yaml` | [budimanjojo.github.io/talhelper](https://budimanjojo.github.io/talhelper/latest/installation/) |
| `kubectl` | Kubernetes CLI | [kubernetes.io/docs](https://kubernetes.io/docs/tasks/tools/) |
| `helm` | Kubernetes package manager | [helm.sh/docs](https://helm.sh/docs/intro/install/) |
| `yq` | YAML processor | [github.com/mikefarah/yq](https://github.com/mikefarah/yq#install) |

## Repository Structure

```
funland/                           # Cluster name
├── argocd/argocd/                 # ArgoCD + ApplicationSet
├── kube-system/cilium/            # Cilium CNI configuration
├── gateway/gateway/               # Gateway API resources (internal/external)
├── gateway-api/gateway-api-crds/  # Gateway API CRD definitions
├── cert-manager/cert-manager/     # TLS certificate management
├── certificate/certificate/       # Wildcard certificates
├── external-secrets/              # External Secrets Operator + 1Password
├── cnpg/                          # CloudNativePG operator + database clusters
├── longhorn-system/longhorn/      # Persistent storage
├── monitoring/                    # Prometheus, Alertmanager, Grafana
├── talos/                         # Talos/Kubernetes versions + node config (talhelper, not ArgoCD)
└── [namespace]/[app]/             # Application deployments
archive/                           # Retired manifests - not deployed (see archive/README.md)
.github/                           # CI validation, Renovate config, PR template
```

### How apps are discovered

The ApplicationSet in `funland/argocd/argocd/appset.yaml` creates one ArgoCD Application for **every directory matching `funland/*/*`**:

- The Application is named after the directory (`funland/media/sonarr` → `sonarr`), so directory names must be unique across the repo.
- The middle path segment is the destination namespace (`media`), created automatically (`CreateNamespace=true`). Resources that need a different namespace must set `metadata.namespace` themselves and declare that Namespace if nothing else does.
- Only files directly inside the directory are applied (no recursion). If the directory contains a `kustomization.yaml`, ArgoCD renders it with Kustomize instead.
- Sync is automated with `prune` and `selfHeal`, so manual `kubectl` edits to managed resources are reverted.

### Adding an application

1. Create `funland/<namespace>/<app>/` with some of:
   - `application.yaml` - a nested ArgoCD `Application` for a Helm chart (Renovate bumps `targetRevision`)
   - `deployment.yaml` / `service.yaml` - raw manifests (Renovate bumps images; keep the `# renovate: image=...` comment above `app.kubernetes.io/version` in sync)
   - `http-route.yaml` - Gateway API `HTTPRoute` attached to the `internal` and/or `external` gateway
   - `external-secret.yaml` - `ExternalSecret`s pulling from the `1password` ClusterSecretStore
   - `README.md` - what it is, dependencies, and the 1Password items it expects
2. Set `resources.requests` on every container (see [Node stuck in NotReady](#node-stuck-in-notready) for why).
3. Add the app to the [Deployed Applications](#deployed-applications) table.
4. Open a PR; CI validates YAML syntax and formatting, Kubernetes schemas, and Kustomize builds.

## Cluster Bootstrap

### 1. Talos Setup

`funland/talos/talconfig.yaml` is the source of truth for the nodes, Talos/Kubernetes versions, system extensions and machine-config patches (including disabling the default CNI and kube-proxy, which Cilium replaces). Generate the machine configs with [talhelper](https://github.com/budimanjojo/talhelper):

```sh
cd funland/talos

# One-time: generate cluster secrets (gitignored - never commit in plaintext)
talhelper gensecret > talsecret.yaml

# Render per-node machine configs + talosconfig into clusterconfig/ (gitignored)
talhelper genconfig
```

Apply the configs and bootstrap the control plane (`192.168.10.20`). `talhelper gencommand` prints the exact per-node commands:

```sh
# Apply each node's config (nodes boot into maintenance mode, hence --insecure)
talhelper gencommand apply --extra-flags="--insecure" | sh

# Bootstrap etcd on the control plane (wait for it to be reachable first)
talhelper gencommand bootstrap | sh

# Fetch a kubeconfig
talhelper gencommand kubeconfig | sh
```

> **Note:** Nodes will show as `NotReady` until Cilium is installed. This is expected since no CNI is configured.

### 2. Install Gateway API CRDs

Install the Gateway API CRDs before Cilium (Cilium's Gateway API integration requires these). This applies the same Kustomization ArgoCD later takes over, so the pinned version and the TLSRoute `v1alpha2` patch match exactly:

```sh
kubectl apply --server-side -k funland/gateway-api/gateway-api-crds/crds
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

### 4. Create the 1Password Connect bootstrap secrets

Every `ExternalSecret` depends on 1Password Connect, which itself needs two secrets that cannot come from GitOps. Create them before ArgoCD starts syncing, following [funland/external-secrets/1password-connect/README.md](funland/external-secrets/1password-connect/README.md):

- `op-connect-credentials` in namespace `1password-connect` (the `1password-credentials.json` file)
- `op-connect-token` in namespace `external-secrets` (the Connect API token)

The 1Password items each app reads are listed in that app's `README.md`.

### 5. Install ArgoCD

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
| Alertmanager ntfy bridge | monitoring | Relays Alertmanager webhooks to ntfy push notifications |
| Authentik Proxy Outpost | authentik | Shared forward-auth SSO proxy for apps with no native OIDC support |
| Cert-Manager | cert-manager | TLS certificate automation |
| Certificate | certificate | Wildcard TLS certificate |
| Cilium | kube-system | CNI with Gateway API, L2 announcements, kube-proxy replacement |
| CloudNativePG | cnpg-system / postgres | PostgreSQL operator (`cnpg-system`) + database clusters (`postgres`) |
| Cloudflare Tunnel | cloudflare | Cloudflare tunnel ingress controller |
| External Secrets Operator | external-secrets | Syncs secrets from 1Password |
| Gateway | gateway | Gateway API Gateway resources (internal/external) |
| Gateway API CRDs | gateway-api | Gateway API CRD definitions |
| Grafana | monitoring | Metrics visualization |
| Immich | immich | Photo/video management |
| Jellyfin | media | Media server |
| Longhorn | longhorn-system | Distributed storage |
| ntfy | ntfy | Push notifications |
| 1Password Connect | 1password-connect | 1Password Connect server (ESO backend) |
| Overseerr | media | Media request management UI (Seerr image, exposed as request.batlab.io) |
| Prometheus | monitoring | Metrics collection + Alertmanager (kube-prometheus-stack) |
| Prowlarr | media | Indexer manager (SSO via Authentik proxy outpost) |
| Radarr | media | Movie collection manager (SSO via Authentik proxy outpost) |
| Recyclarr | media | Syncs TRaSH Guides quality profiles into Radarr/Sonarr |
| Sonarr | media | TV collection manager (SSO via Authentik proxy outpost) |
| Vaultwarden | vaultwarden | Bitwarden-compatible password manager |

## Network Architecture

| Address | What |
|---------|------|
| `192.168.10.20` | Control plane (`talos-g4c-g4l`), Kubernetes API on `:6443` |
| `192.168.10.21` - `.23` | Workers (`talos-worker1`-`3`) |
| `192.168.10.30` - `.49` | Cilium LoadBalancer IP pool (L2 announcements / ARP) |
| `192.168.10.30` | Internal Gateway - routes internal services via HTTPRoute |
| `192.168.10.31` | External Gateway - routes public-facing services |
| `192.168.10.35` | AdGuard Home DNS (53/tcp+udp, 853/tcp) |
| `192.168.11.13` | NFS server for media and the Immich library (outside the cluster) |

- **CNI:** Cilium with kube-proxy replacement, L2 announcements, and Gateway API enabled
- **TLS:** a single `*.batlab.io` wildcard certificate (cert-manager, Let's Encrypt DNS-01 via Cloudflare) in the `certificate` namespace, shared by both gateways
- **Cloudflare Tunnel:** `auth`, `immich` and `ntfy` are also published through the Cloudflare Tunnel ingress controller (`ingress.yaml` in those apps)

## Observability Stack

### Metrics (Prometheus + Grafana)
- Prometheus scrapes every ServiceMonitor/PodMonitor in the cluster (selectors are unrestricted)
- Cilium agent, operator and Envoy metrics are enabled; Hubble (flow/DNS/HTTP metrics and UI) is currently **disabled** - see `funland/kube-system/cilium/README.md`
- Grafana dashboards available at `grafana.batlab.io`

### Alerting
- Prometheus → Alertmanager → `alertmanager-ntfy-bridge` → ntfy topic `alerts`
- The `Watchdog` alert is sent to an external dead-man's-switch so a failure of the in-cluster alert path is itself noticed (see `funland/monitoring/prometheus/external-secret-watchdog.yaml`)

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
- **Schedule:** Daily at midnight UTC (via ScheduledBackup resources)
- **Continuous:** WAL archiving enabled for point-in-time recovery

**Verify backups:**
```sh
kubectl get scheduledbackups -n postgres
kubectl get backups -n postgres
```

## Upgrading Talos & Kubernetes

`funland/talos/talconfig.yaml` ([talhelper](https://github.com/budimanjojo/talhelper) config) is the source of truth for the Talos OS version (`talosVersion`) and the Kubernetes version (`kubernetesVersion`). It also declares the Longhorn-required Talos system extensions (`siderolabs/iscsi-tools`, `siderolabs/util-linux-tools`) applied to every node.

Renovate does **not** watch this file (see `.github/renovate.json`, which explicitly ignores `funland/talos/**`) — it only manages the apps running in the cluster (Helm charts, Deployments, images), never the cluster/OS layer itself. Watch [Talos releases](https://github.com/siderolabs/talos/releases) and the [Kubernetes changelog](https://github.com/kubernetes/kubernetes/releases) yourself and bump `talconfig.yaml` manually when you're ready to upgrade.

Argo CD is not involved in this process either: it only syncs Kubernetes API resources, while Talos/Kubernetes core upgrades go through the separate Talos API (`talosctl`). Bumping `talosVersion`/`kubernetesVersion` in `talconfig.yaml` just records the target version — applying it is a manual runbook:

```sh
# Regenerate machine configs from the updated talconfig.yaml.
# This also resolves the current schematic (Longhorn extensions) into an
# installer image reference for each node, written to clusterconfig/*.yaml.
talhelper genconfig

# Look up the resolved installer image for a node, e.g.:
yq '.machine.install.image' clusterconfig/funland-talos-worker1.yaml
# -> factory.talos.dev/installer/<schematic-id>:<new-talos-version>

# Roll the Talos OS upgrade node-by-node - workers first, control plane last.
# Validate cluster/Longhorn health between each node before continuing.
talosctl upgrade --nodes 192.168.10.21 --image <installer-image-from-above>
talosctl upgrade --nodes 192.168.10.22 --image <installer-image-from-above>
talosctl upgrade --nodes 192.168.10.23 --image <installer-image-from-above>
talosctl upgrade --nodes 192.168.10.20 --image <installer-image-from-above>

# Once every node is on the new Talos version, bump Kubernetes components
talosctl upgrade-k8s --nodes 192.168.10.20 --to <new-kubernetes-version>
```

> **Note:** `talhelper` also needs a `talsecret.yaml` (cluster/node secrets) to generate machine configs. Generate it locally with `talhelper gensecret` — it's gitignored and must never be committed in plaintext.

## Troubleshooting

### Node stuck in NotReady

First check Cilium — if the CNI isn't running the node never goes Ready:
`kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium`

If Cilium is healthy and nodes are *flapping* rather than staying down, suspect
memory accounting. Compare what the scheduler thinks is allocated against what
the node is really using:

```sh
kubectl describe node <node> | grep -A6 "Allocated resources"
kubectl top nodes
```

If "Requests" for memory is far below actual usage, the scheduler is packing
pods onto a node it believes is empty, and the kernel OOM killer will eventually
reboot it. That was the cause of the 2026-08-23 incident (worker2 reported 7%
memory requested while running at 71%, and took 153 kernel OOM kills in a week).
Any workload with no `resources.requests` counts as zero to the scheduler.

Confirm OOM kills and reboots directly:

```sh
# kernel OOM kills per node over the last 24h - should be 0
kubectl get --raw "/api/v1/namespaces/monitoring/services/prometheus-application-kub-prometheus:9090/proxy/api/v1/query?query=increase(node_vmstat_oom_kill%5B24h%5D)"

# node uptime - a reset here means a real reboot, not a network blip
kubectl get --raw "/api/v1/namespaces/monitoring/services/prometheus-application-kub-prometheus:9090/proxy/api/v1/query?query=time()-node_boot_time_seconds"
```

See `funland/monitoring/prometheus/README.md` for the full write-up.

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
