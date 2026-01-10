# CLAUDE.md - Claude Code Configuration

## Project Overview

This is a GitOps-managed Kubernetes configuration repository for a home lab cluster running **Talos Linux**. All infrastructure is declarative and managed via **ArgoCD ApplicationSets**.

## Key Technologies

- **Talos Linux**: Immutable, secure Kubernetes OS
- **ArgoCD**: GitOps continuous deployment with ApplicationSet pattern
- **Cilium**: CNI with Gateway API, L2 announcements, kube-proxy replacement
- **CloudNativePG**: PostgreSQL operator for HA database clusters
- **External Secrets + 1Password Connect**: Secrets management pipeline
- **Cert-Manager**: TLS automation with Let's Encrypt + Cloudflare DNS
- **Longhorn**: Distributed block storage
- **Helm + Kustomize**: Package management

## Repository Structure

```
funland/                          # Cluster name
├── [namespace]/[app]/            # Standard app pattern
│   ├── application.yaml          # ArgoCD Application (Helm charts)
│   ├── namespace.yaml            # Namespace definition (raw manifests)
│   ├── deployment.yaml           # Deployment only (raw manifests)
│   ├── service.yaml              # Service definition
│   ├── http-route.yaml           # Gateway API routing
│   ├── pvc.yaml                  # PersistentVolumeClaim
│   ├── external-secret.yaml      # 1Password integration
│   ├── configmap.yaml            # ConfigMap (if needed)
│   └── kustomization.yaml        # Kustomize overlays (if needed)
```

The ApplicationSet in `funland/argocd/argocd/appset.yaml` auto-discovers apps from `funland/*/*`.

## Common Patterns

### Adding a New Helm Application

Create `funland/[namespace]/[app-name]/application.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-name
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.example.com
    chart: chart-name
    targetRevision: "1.0.0"
    helm:
      valuesObject:
        # Helm values here
  destination:
    server: https://kubernetes.default.svc
    namespace: app-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Adding Secrets from 1Password

Create an ExternalSecret that references the ClusterSecretStore:
```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-secret
  namespace: app-namespace
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: 1password
    kind: ClusterSecretStore
  target:
    name: app-secret
    creationPolicy: Owner
  data:
    - secretKey: PASSWORD
      remoteRef:
        key: vault-item-name
        property: password
```

### Adding HTTP Routes (Ingress)

Use Gateway API HTTPRoute pointing to either the internal or external gateway:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
  namespace: app-namespace
spec:
  parentRefs:
    - name: internal      # Use "external" for public-facing services
      namespace: gateway
      sectionName: https
  hostnames:
    - app.batlab.io
  rules:
    - backendRefs:
        - name: app-service
          port: 8080
```

**Gateway options:**
- `internal` - For services only accessible on the local network
- `external` - For services exposed publicly (used by: jellyfin, ntfy, privatebin, synapse, vikunja)

### Adding a CloudNativePG Database

Add cluster definition to `funland/cnpg/cnpg-clusters/`:
```yaml
apiVersion: cnpg.io/v1
kind: Cluster
metadata:
  name: app-dbc
  namespace: app-namespace
spec:
  instances: 3
  storage:
    size: 5Gi
    storageClass: longhorn
  bootstrap:
    initdb:
      database: app_db
      owner: app_user
      secret:
        name: app-db-credentials
  backup:
    barmanObjectStore:
      destinationPath: s3://bucket/backups/app-namespace/app-dbc
      # ... S3 configuration
  monitoring:
    enablePodMonitor: true
```

## Important Files

| File | Purpose |
|------|---------|
| `funland/argocd/argocd/appset.yaml` | Main ApplicationSet that discovers all apps |
| `funland/gateway/gateway/internal-gateway.yaml` | Internal Gateway for local network services |
| `funland/gateway/gateway/external-gateway.yaml` | External Gateway for public-facing services |
| `funland/external-secrets/cluster-secret-store/` | 1Password ClusterSecretStore |
| `funland/certificate/certificate/wildcard.yaml` | Wildcard cert for *.batlab.io |
| `funland/cnpg/cnpg-clusters/` | All PostgreSQL cluster definitions |

## Network Configuration

- **IP Pool**: 192.168.10.30-192.168.10.49 (Cilium L2)
- **Internal Gateway**: 192.168.10.30 (local network only)
- **External Gateway**: 192.168.10.31 (public-facing services)
- **Domain**: *.batlab.io (wildcard TLS)
- **DNS Validation**: Cloudflare

## Best Practices for This Repo

1. **Never use `kubectl apply` directly** - All changes go through Git and ArgoCD
2. **Use ExternalSecret for all credentials** - Never commit secrets to Git
3. **Follow the namespace/app directory pattern** - ArgoCD discovers apps automatically
4. **Use Helm when available** - Prefer Helm charts over raw manifests for maintainability
5. **CloudNativePG for databases** - Don't deploy standalone PostgreSQL pods
6. **HTTPRoute for ingress** - Use Gateway API, not Ingress resources
7. **Longhorn for persistent storage** - Default storageClass for PVCs

## Renovate Integration

Renovate automatically updates:
- Helm chart versions in `application.yaml` files
- Container image tags matching specific patterns

Check `.github/renovate.json` for custom regex patterns.

## Debugging Commands

```bash
# Check ArgoCD app status
kubectl get applications -n argocd

# View app sync status
kubectl get applications -n argocd -o wide

# Check CloudNativePG clusters
kubectl get clusters -A

# View External Secrets sync status
kubectl get externalsecrets -A

# Check Gateway API routes
kubectl get httproutes -A

# View Cilium status
cilium status
```

## When Making Changes

1. Create/modify files in the appropriate `funland/[namespace]/[app]/` directory
2. Commit and push to the main branch
3. ArgoCD will automatically sync changes (prune + self-heal enabled)
4. Monitor sync status in ArgoCD UI or via kubectl

## Testing Changes Locally

Use `kubectl diff` or ArgoCD's diff feature before committing:
```bash
# Dry-run a Helm template
helm template [release-name] [chart] -f values.yaml

# Check Kustomize output
kubectl kustomize funland/[namespace]/[app]/
```
