# CloudNativePG Backblaze B2 Backup Configuration

This directory contains the configuration for backing up PostgreSQL databases to Backblaze B2 using CloudNativePG's built-in backup capabilities.

## Architecture

The backup solution uses:
- **CloudNativePG (CNPG)**: PostgreSQL operator with built-in backup support
- **Backblaze B2**: S3-compatible object storage for backup destination
- **Barman**: PostgreSQL backup tool (integrated into CNPG)
- **External Secrets Operator**: Pulls B2 credentials from 1Password
- **Kustomize Components**: Reusable backup configuration template

## Directory Structure

```
funland/cnpg/
├── components/
│   └── b2-backup/              # Reusable backup configuration component
│       ├── backup-config.yaml  # Strategic merge patch for backup config
│       └── kustomization.yaml  # Component definition
├── external-secrets/
│   └── b2-backup-credentials.yaml  # ExternalSecret for B2 credentials
└── cnpg-clusters/
    ├── kustomization.yaml      # Applies backup to all clusters
    ├── authentik-cnpg-cluster.yaml
    ├── vikunja-cnpg-cluster.yaml
    ├── n8n-cnpg-cluster.yaml
    ├── vaultwarden-cnpg-cluster.yaml
    └── immich-cnpg-cluster.yaml
```

## Setup Instructions

### 1. Configure Backblaze B2

1. Create a B2 bucket named `batcave-kubernetes` (or update the bucket name in the config)
2. Create an application key with read/write access to the bucket
3. Note your B2 credentials:
   - **keyId**: Your application key ID
   - **applicationKey**: Your application key (secret)

### 2. Store Credentials in 1Password

Create a new item in 1Password with the following fields:
- **Item name**: `b2-backup-credentials`
- **Field name**: `keyId` → Your B2 application key ID
- **Field name**: `applicationKey` → Your B2 application key

The ExternalSecret in `external-secrets/b2-backup-credentials.yaml` will pull these credentials and create a Kubernetes secret.

### 3. Deploy the Configuration

The backup configuration is automatically applied to all CNPG clusters via Kustomize:

```bash
# Build and preview the configuration
kubectl kustomize funland/cnpg/cnpg-clusters/

# Apply via kubectl
kubectl apply -k funland/cnpg/cnpg-clusters/

# Or let ArgoCD sync it automatically
```

### 4. Verify Backup Configuration

Check that the secret was created:
```bash
kubectl get secret b2-backup-credentials -n postgres
```

Check cluster status:
```bash
kubectl get cluster -n postgres
kubectl describe cluster authentik-dbc -n postgres
```

## Backup Configuration Details

### Compression Options

The backup configuration uses **snappy** compression by default. You can change this in `cnpg-components/b2-backup/backup-config.yaml`:

**Available compression algorithms:**
- `snappy`: Fast compression, moderate compression ratio (recommended for most cases)
- `gzip`: Good compression, slower than snappy (good for storage-constrained environments)
- `bzip2`: Best compression, slowest (rarely used)
- `zstd`: Excellent compression and speed (requires CNPG 1.23+)

**To use gzip instead:**
```yaml
wal:
  compression: gzip
data:
  compression: gzip
```

**To use zstd (requires CNPG 1.23+):**
```yaml
wal:
  compression: zstd
data:
  compression: zstd
```

### Retention Policy

Backups are retained for **30 days** by default. Change in `cnpg-components/b2-backup/backup-config.yaml`:
```yaml
backup:
  retentionPolicy: "30d"  # Format: <number>d (days), <number>w (weeks), <number>m (months)
```

### Encryption

Server-side encryption (AES256) is enabled by default:
```yaml
wal:
  encryption: AES256
data:
  encryption: AES256
```

## Backup Operations

### Trigger a Manual Backup

Create a `Backup` resource:

```bash
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: authentik-backup-$(date +%Y%m%d-%H%M%S)
  namespace: postgres
spec:
  cluster:
    name: authentik-dbc
  method: barmanObjectStore
EOF
```

### List Backups

```bash
kubectl get backups -n postgres
```

### Check Backup Status

```bash
kubectl describe backup <backup-name> -n postgres
```

### View Cluster Backup Status

```bash
kubectl get cluster authentik-dbc -n postgres -o jsonpath='{.status.lastSuccessfulBackup}'
```

## Scheduled Backups

CNPG automatically performs continuous WAL archiving. To add scheduled full backups, create a `ScheduledBackup` resource:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: authentik-daily-backup
  namespace: postgres
spec:
  schedule: "0 0 2 * * *"  # Daily at 2 AM
  backupOwnerReference: self
  cluster:
    name: authentik-dbc
  method: barmanObjectStore
```

Apply it:
```bash
kubectl apply -f scheduled-backup.yaml
```

## Restore Operations

### Restore to a New Cluster

Create a new cluster that bootstraps from a backup:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: authentik-dbc-restored
  namespace: postgres
spec:
  instances: 1
  storage:
    size: 1Gi

  bootstrap:
    recovery:
      source: authentik-dbc-backup

  externalClusters:
    - name: authentik-dbc-backup
      barmanObjectStore:
        destinationPath: s3://batcave-kubernetes/funland/postgres-backups/authentik
        endpointURL: https://s3.us-west-000.backblazeb2.com
        s3Credentials:
          accessKeyId:
            name: b2-backup-credentials
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: b2-backup-credentials
            key: ACCESS_SECRET_KEY
        wal:
          compression: snappy
        s3:
          region: us-west-000
```

### Point-in-Time Recovery (PITR)

Restore to a specific timestamp:

```yaml
bootstrap:
  recovery:
    source: authentik-dbc-backup
    recoveryTarget:
      targetTime: "2024-01-15 10:00:00.00000+00"
```

## Monitoring

### Check WAL Archiving

```bash
# View cluster status
kubectl get cluster -n postgres

# Check detailed status
kubectl describe cluster authentik-dbc -n postgres | grep -A 10 "Continuous Archiving"
```

### View Backup Logs

```bash
# Get pod name
kubectl get pods -n postgres -l cnpg.io/cluster=authentik-dbc

# View logs
kubectl logs -n postgres <pod-name> | grep backup
```

## Backup Paths in B2

Backups are organized in B2 as follows:
```
batcave-kubernetes/
└── funland/
    └── postgres-backups/
        ├── authentik/
        ├── vikunja/
        ├── n8n/
        ├── vaultwarden/
        └── immich/
```

Each database directory contains:
- `base/`: Full database backups
- `wals/`: Write-Ahead Log files for point-in-time recovery

## Cost Optimization

**Storage costs** (Backblaze B2):
- Storage: $0.005/GB/month
- Download: $0.01/GB
- API calls: Free for first 2,500 downloads/day

**Tips:**
1. Adjust retention policy based on compliance needs
2. Use snappy compression for good balance of speed and size
3. Monitor backup sizes: `kubectl get backup -n postgres`
4. Consider lifecycle policies in B2 for older backups

## Troubleshooting

### Backup Fails with "AccessDenied"

Check that your B2 application key has permissions for the bucket:
```bash
kubectl get secret b2-backup-credentials -n postgres -o jsonpath='{.data.ACCESS_KEY_ID}' | base64 -d
```

### WAL Archiving Not Working

Check cluster status:
```bash
kubectl describe cluster authentik-dbc -n postgres
```

Check pod logs:
```bash
kubectl logs -n postgres authentik-dbc-1 | grep -i wal
```

### Cannot Access Backups

Verify the endpoint URL and region match your B2 bucket settings. For us-west-000, the endpoint should be:
```
https://s3.us-west-000.backblazeb2.com
```

## Adding Backups to a New Cluster

When creating a new CNPG cluster, simply add it to `cnpg-clusters/kustomization.yaml`:

1. Create your cluster YAML file (e.g., `newapp-cnpg-cluster.yaml`)
2. Add it to the `resources` list in `kustomization.yaml`
3. Add a patch with the specific backup path

The backup component will automatically be applied!

## References

- [CloudNativePG Backup Documentation](https://cloudnative-pg.io/documentation/current/backup_recovery/)
- [Backblaze B2 S3 Compatible API](https://www.backblaze.com/b2/docs/s3_compatible_api.html)
- [Barman Documentation](https://pgbarman.org/)
