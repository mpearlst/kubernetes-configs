# cert-manager

This directory contains the manifests for deploying [cert-manager](https://cert-manager.io/).

cert-manager is a native Kubernetes certificate management controller. It can help with issuing certificates from a variety of sources, such as Let's Encrypt, HashiCorp Vault, Venafi, a simple signing key pair, or self signed.

## Helm Chart

This deployment uses the `cert-manager` Helm chart from the [Jetstack chart repository](https://charts.jetstack.io).

## Configuration

This deployment is configured to use Let's Encrypt to automatically issue SSL certificates. It uses Cloudflare for the DNS-01 challenge to verify domain ownership.

## Dependencies

This application has the following dependencies:

- **[Let's Encrypt](https://letsencrypt.org/):** Used as the certificate authority to issue SSL certificates.
- **[Cloudflare](https://www.cloudflare.com/):** Used as the DNS provider for the DNS-01 challenge.
- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to fetch the Cloudflare API token from 1Password.

## Secrets

This application requires a Cloudflare API token for the DNS-01 challenge. The token is stored in a Kubernetes secret with the following details:

- **Secret Name:** `cloudflare-api-token-secret`
- **Key:** `cloudflare-api-key`

This secret is created using External Secrets, which fetches the token from a 1Password secret named `cloudflare` with the property `cloudflare-api-token-secret`.
