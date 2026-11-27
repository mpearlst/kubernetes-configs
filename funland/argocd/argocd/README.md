# Argo CD

This directory contains the manifests for deploying [Argo CD](https://argo-cd.readthedocs.io/en/stable/).

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

## Helm Chart

This deployment uses the `argo-cd` Helm chart from the [argo-helm repository](https://argoproj.github.io/argo-helm).

## Dependencies

This application has the following dependencies:

- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to fetch credentials from 1Password for OIDC integration.
- **[Authentik](https://goauthentik.io/):** It is configured to use Authentik for OIDC-based authentication.
- **Redis:** It uses a Redis instance for caching, which is deployed as part of the Helm chart.

## Secrets

This application requires the following secrets for OIDC integration with Authentik:

- **Secret Name:** `argocd-sso`
- **Keys:**
  - `client_id`
  - `client_secret`

These secrets are fetched from a 1Password secret named `argocd-sso` with the following properties:

- `argocd_client_id`
- `argocd_client_secret`

## ApplicationSet

`appset.yaml` defines the `argocd-appset` ApplicationSet that turns every `funland/<namespace>/<app>/` directory into an ArgoCD Application named `<app>`, deployed to `<namespace>`, with automated sync, prune and self-heal. See "How apps are discovered" in the root `README.md`.

It also carries the cluster-wide `ignoreDifferences` for CNPG `Cluster` fields that the operator defaults server-side, which would otherwise show as permanently OutOfSync.
