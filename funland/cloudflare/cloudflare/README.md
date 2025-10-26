# Cloudflare Tunnel Ingress Controller

This directory contains the manifests for deploying a [Cloudflare Tunnel Ingress Controller](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-with-ingress-controller/).

This controller allows you to expose services running in your Kubernetes cluster to the internet via Cloudflare Tunnels, without opening inbound ports on your firewall.

## Helm Chart

This deployment uses the `cloudflare-tunnel-ingress-controller` Helm chart from the [strrl.dev Helm repository](https://helm.strrl.dev).

## Image

This deployment uses the `cloudflared` Docker image.
## Dependencies

This application has the following dependencies:

- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to fetch Cloudflare API token, tunnel name, and account ID from 1Password.
- **Cloudflare:** It relies on the Cloudflare Tunnel service to establish secure connections.

## Secrets

This application requires a secret named `cloudflare-operator-secret` with the following keys:

- `cloudflare-operator-api-token-secret`
- `cloudflare-operator-tunnel-name-secret`
- `cloudflare-operator-account-id-secret`

This secret is created using External Secrets, which fetches the values from a 1Password secret named `cloudflare-operator` with the following properties:

- `password` (maps to `cloudflare-operator-api-token-secret`)
- `tunnelName` (maps to `cloudflare-operator-tunnel-name-secret`)
- `accountId` (maps to `cloudflare-operator-account-id-secret`)
