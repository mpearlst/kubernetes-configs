# ClusterSecretStore

Defines the `ClusterSecretStore` named `1password` that External Secrets Operator uses to fetch secrets from 1Password vaults via the Connect API.

## Prerequisites

The Connect API token secret must exist in the `external-secrets` namespace before this store can become `Ready`. See the [1password-connect bootstrap instructions](../1password-connect/README.md) for how to create it.

- **Secret name:** `op-connect-token`
- **Namespace:** `external-secrets`
- **Key:** `token`

## How it works

ESO reads `op-connect-token` to authenticate HTTP requests to the Connect server at `http://onepassword-connect.1password-connect.svc.cluster.local:8080`. All `ExternalSecret` resources that reference `secretStoreRef.name: 1password` go through this store.

## Verification

```bash
kubectl get clustersecretstore 1password
```

Status should show `Ready: True`. If it shows `ConnectionError`, check that:
1. The `op-connect-token` secret exists in the `external-secrets` namespace
2. The Connect pods are running in `1password-connect`
3. The token value is correct
