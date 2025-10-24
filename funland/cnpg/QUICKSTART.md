# B2 Backup Quick Start Guide

## What Was Created

The backup configuration has been set up with the following structure:

```
funland/cnpg/
├── kustomization.yaml                    # Parent kustomization
├── components/
│   └── b2-backup/                        # Reusable backup template
│       ├── backup-config.yaml            # Backup configuration
│       └── kustomization.yaml            # Component definition
├── external-secrets/
│   └── b2-backup-credentials.yaml        # B2 credentials from 1Password
└── cnpg-clusters/
    ├── kustomization.yaml                # Applies backup to all clusters
    ├── scheduled-backups.yaml            # Daily backups at midnight
    └── [cluster files...]                # Your existing cluster definitions
```

## Setup Steps

### 1. Create B2 Credentials in 1Password

In your 1Password vault, create an item named `b2-backup-credentials` with:
- **keyId**: Your Backblaze application key ID
- **applicationKey**: Your Backblaze application key (secret)

### 2. Apply the Configuration

```bash
# Preview what will be created
kubectl kustomize funland/cnpg/

# Apply the configuration
kubectl apply -k funland/cnpg/
```

### 3. Verify

```bash
# Check the B2 credentials secret was created
kubectl get secret b2-backup-credentials -n postgres

# Check cluster status
kubectl get cluster -n postgres

# Check scheduled backups
kubectl get scheduledbackup -n postgres

# Verify backup configuration on a cluster
kubectl get cluster authentik-dbc -n postgres -o yaml | grep -A 20 backup
```

## Backup Details

### Configuration Applied to All Clusters:
- **Destination**: `s3://batcave-kubernetes/funland/postgres-backups/{database-name}`
- **Endpoint**: `https://s3.us-west-000.backblazeb2.com`
- **Region**: `us-west-000`
- **Compression**: `snappy` (fast, good balance)
- **Encryption**: `AES256` (server-side)
- **Retention**: `30 days`
- **Schedule**: Daily at `12:00 AM` (midnight)

### Backups Created:
1. **Continuous WAL archiving** - Real-time transaction log backups (automatic)
2. **Scheduled full backups** - Daily at midnight (ScheduledBackup resources)
3. **Base backups** - On-demand backups when requested

## Common Operations

### Trigger a Manual Backup

```bash
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: authentik-manual-$(date +%Y%m%d-%H%M%S)
  namespace: postgres
spec:
  cluster:
    name: authentik-dbc
  method: barmanObjectStore
EOF
```

### List All Backups

```bash
kubectl get backups -n postgres
```

### View Backup Status

```bash
kubectl describe backup <backup-name> -n postgres
```

### Check Last Successful Backup

```bash
kubectl get cluster -n postgres -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.lastSuccessfulBackup}{"\n"}{end}'
```

## Compression Options

The default compression is **snappy** for optimal performance. To change:

Edit `funland/cnpg/cnpg-components/b2-backup/backup-config.yaml`:

**For better compression (slower):**
```yaml
wal:
  compression: gzip  # or bzip2
data:
  compression: gzip  # or bzip2
```

**For zstd (CNPG 1.23+):**
```yaml
wal:
  compression: zstd
data:
  compression: zstd
```

## Adding Backups to New Clusters

When you create a new CNPG cluster:

1. Create your cluster YAML in `funland/cnpg/cnpg-clusters/`
2. Add it to `resources` in `funland/cnpg/cnpg-clusters/kustomization.yaml`
3. Add a patch for the custom destination path:

```yaml
- target:
    kind: Cluster
    name: newapp-dbc
  patch: |-
    - op: replace
      path: /spec/backup/barmanObjectStore/destinationPath
      value: s3://batcave-kubernetes/funland/postgres-backups/newapp
    - op: add
      path: /spec/backup/barmanObjectStore/tags/cluster
      value: newapp-dbc
```

4. (Optional) Add a ScheduledBackup in `scheduled-backups.yaml`

The backup component will automatically be applied!

## Monitoring

### Check Cluster Health
```bash
kubectl get cluster -n postgres
```

### View Cluster Details
```bash
kubectl describe cluster authentik-dbc -n postgres
```

### Check WAL Archiving
```bash
kubectl get cluster authentik-dbc -n postgres -o jsonpath='{.status.continuousArchiving}'
```

### View Logs
```bash
# Get pod name
kubectl get pods -n postgres -l cnpg.io/cluster=authentik-dbc

# View logs
kubectl logs -n postgres <pod-name> | grep -i backup
```

## Troubleshooting

### Secret Not Found
Ensure the ExternalSecret synced:
```bash
kubectl get externalsecret b2-backup-credentials -n postgres
kubectl describe externalsecret b2-backup-credentials -n postgres
```

### Backup Fails
Check cluster events:
```bash
kubectl describe cluster authentik-dbc -n postgres
```

Check backup logs:
```bash
kubectl describe backup <backup-name> -n postgres
```

### Permission Denied
Verify B2 credentials have permissions for the bucket:
```bash
kubectl get secret b2-backup-credentials -n postgres -o yaml
```

## Cost Estimates

**Backblaze B2 Pricing (as of 2024):**
- Storage: $0.005/GB/month
- Download: $0.01/GB
- API calls: First 2,500 downloads/day free

**Example monthly cost for 5 databases with 10GB total backups:**
- Storage: 10GB × $0.005 = $0.05/month
- WAL archiving API calls: Included in free tier
- Downloads (restores): $0.01/GB when needed

## Documentation

For detailed information, see:
- `README-BACKUPS.md` - Comprehensive documentation
- [CloudNativePG Docs](https://cloudnative-pg.io/documentation/current/backup_recovery/)
- [Backblaze B2 S3 API](https://www.backblaze.com/b2/docs/s3_compatible_api.html)
