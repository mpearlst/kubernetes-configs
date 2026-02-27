# 1Password Connect

Deploys the [1Password Connect server](https://developer.1password.com/docs/connect/) via the official Helm chart. The Connect server is the bridge between the cluster and 1Password vaults — all `ExternalSecret` resources depend on it being healthy.

## Dependency chain

```
[Manual bootstrap secrets]
        ↓
1Password Connect pods start  (uses op-connect-credentials)
        ↓
ClusterSecretStore becomes Ready  (uses op-connect-token to call Connect API)
        ↓
All ExternalSecrets sync
```

## Bootstrap (one-time manual setup)

These two secrets cannot be managed by GitOps because they are prerequisites for GitOps itself. Create them once before ArgoCD syncs this app.

**1. Connect server credentials** — the `1password-credentials.json` file downloaded from 1Password when you created the Connect server integration:

```bash
kubectl create namespace 1password-connect

kubectl create secret generic op-connect-credentials \
  --from-file=1password-credentials.json=./1password-credentials.json \
  -n 1password-connect
```

**2. Connect API token** — the token used by External Secrets Operator to authenticate to the Connect API:

```bash
kubectl create secret generic op-connect-token \
  --from-literal=token=<your-connect-api-token> \
  -n external-secrets
```

Both values come from the 1Password Connect server setup in your 1Password account under **Integrations → Connect Server**.

## After bootstrap

Once the secrets exist, ArgoCD will:
1. Deploy 1Password Connect into the `1password-connect` namespace
2. The ClusterSecretStore will connect to Connect at `http://onepassword-connect.1password-connect.svc.cluster.local:8080`
3. All `ExternalSecret` resources across the cluster will begin syncing

## Verification

```bash
kubectl get pods -n 1password-connect
kubectl get clustersecretstore 1password
kubectl get externalsecrets -A
```

## Notes

- Helm release name is `onepassword`, which makes the Connect service DNS name `onepassword-connect` — this must match the `connectHost` in `cluster-secret-store.yaml`
- The `CreateNamespace=true` sync option creates the `1password-connect` namespace automatically; the bootstrap step above creates it first so the secret can be placed there before ArgoCD runs
- Renovate manages `targetRevision` bumps automatically
