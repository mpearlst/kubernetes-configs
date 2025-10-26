# 1Password Connect

This directory contains the manifests for deploying the [1Password Connect server](https://developer.1password.com/docs/connect/).

The 1Password Connect server allows applications and services to securely access items stored in 1Password vaults.

## Helm Chart

This deployment uses the `connect` Helm chart from the [1Password Helm charts repository](https://1password.github.io/connect-helm-charts).
## Dependencies

This application has the following dependencies:

- **[External Secrets Operator](https://external-secrets.io/):** It uses an ExternalSecret to fetch the 1Password credentials JSON required by the Connect server from 1Password itself.
- **1Password Account:** A 1Password account with a service account configured to access the necessary vaults.

## Secrets

This application requires a secret named `1passwordconnect` containing the `1password-credentials.json` file. This secret is created using External Secrets, which fetches the JSON content from a 1Password secret named `1password` with the property `1password-credentials.json`.

- **Secret Name:** `1passwordconnect`
- **Key:** `1password-credentials.json`
