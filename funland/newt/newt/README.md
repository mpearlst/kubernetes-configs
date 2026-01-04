# Newt

This directory contains the manifests for deploying [Newt](https://github.com/fosrl/newt), a Pangolin tunnel client.

Newt is a WireGuard-based tunnel client that connects to a Pangolin server to establish secure tunnels for exposing services.

## Helm Chart

This deployment uses the `newt` Helm chart from the [fosrl Helm repository](https://fosrl.github.io/helm-charts).

## Dependencies

This application has the following dependencies:

- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to fetch Pangolin endpoint, ID, and secret from 1Password.
- **Pangolin Server:** It requires a running Pangolin server to connect to.

## Secrets

This application requires a secret named `newt-auth` with the following keys:

- `PANGOLIN_ENDPOINT`
- `NEWT_ID`
- `NEWT_SECRET`

This secret is created using External Secrets, which fetches the values from a 1Password secret named `newt` with the following properties:

- `endpoint` (maps to `PANGOLIN_ENDPOINT`)
- `id` (maps to `NEWT_ID`)
- `secret` (maps to `NEWT_SECRET`)
