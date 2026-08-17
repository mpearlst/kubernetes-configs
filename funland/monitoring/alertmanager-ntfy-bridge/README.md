# alertmanager-ntfy-bridge

This directory contains the manifests for deploying [alertmanager-ntfy](https://github.com/alexbakker/alertmanager-ntfy), a small relay that translates Prometheus Alertmanager's webhook payloads into [ntfy](https://ntfy.sh/) push notifications.

## Why this exists

Alertmanager's built-in `webhook_configs` receiver posts its own internal JSON schema — pointing it directly at ntfy's publish endpoint produces an unreadable raw-JSON push. This bridge sits in between, in-cluster only (ClusterIP, no ingress), and reformats each alert into a readable ntfy notification (title, description, click-through link to the Prometheus alert).

It's the notification path for `funland/monitoring/prometheus`'s Alertmanager (`alertmanager.config.receivers` in that app's `application.yaml` routes everything except the `Watchdog` heartbeat alert to this bridge) and pushes to the `alerts` topic on the `ntfy` deployment (`funland/ntfy/ntfy`).

## Dependencies

- **[External Secrets](https://external-secrets.io/):** the bridge's `config.yaml` (including the ntfy auth token) is templated by an `ExternalSecret` sourced from 1Password.
- **ntfy:** publishes to `http://ntfy.ntfy.svc.cluster.local` using a token scoped to the `alerts` topic.

## One-time setup (not managed by GitOps)

ntfy's user/token database lives on its own PVC and isn't declarative, so the publishing user and token have to be created once against the running ntfy instance. This only needs to be redone if the `ntfy` PVC is ever wiped/recreated.

`ntfy user add` prompts for a password interactively, which needs a real TTY — a plain `kubectl exec` (no `-it`) fails with `password: inappropriate ioctl for device`. Skip the prompt entirely with the `NTFY_PASSWORD` env var the ntfy CLI checks for before it ever prompts (the password itself is throwaway; only the token from the last command is actually used):

```sh
kubectl exec -n ntfy deploy/ntfy -- sh -c 'NTFY_PASSWORD="$1" ntfy user add --role=user alertmanager-bridge' _ "$(openssl rand -base64 24)"
kubectl exec -n ntfy deploy/ntfy -- ntfy access alertmanager-bridge alerts write-only
kubectl exec -n ntfy deploy/ntfy -- ntfy token add alertmanager-bridge
```

The last command prints a `tk_...` token. Store it in the 1Password item `alertmanager-ntfy-bridge`, property `ntfy_token` (same vault as the existing `ntfy` and `prowlarr` items referenced by other `ExternalSecret`s in this repo) — the `ExternalSecret` in this directory reads it from there.

Subscribe to the `alerts` topic in the ntfy app/UI to actually receive the pushes.

To verify the user/access were set up correctly without needing the token again:

```sh
kubectl exec -n ntfy deploy/ntfy -- ntfy user list
```

**Automating this:** the user-creation and access-grant steps above could be turned into an ArgoCD `PostSync` hook `Job` that execs into the ntfy pod the same way (idempotent — check `ntfy user list` first, skip if the user already exists). That was deliberately not done here: it would need a `ServiceAccount` with `pods/exec` RBAC on the `ntfy` pod, a new privileged capability in the cluster, to automate three commands that only need to run once (or after a PVC wipe). The token step can't be safely automated the same way without also giving the Job `secrets` write access and moving that one secret's source of truth outside 1Password, which every other secret in this repo relies on — see the discussion that led to this decision if revisiting.
