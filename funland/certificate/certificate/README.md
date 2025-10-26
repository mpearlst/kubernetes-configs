# Certificate Configuration

This directory contains Kubernetes resources for managing a wildcard TLS certificate for `*.batlab.io` and `batlab.io`.

## Certificate Resource

The `wildcard.yaml` file defines a `Certificate` resource that instructs cert-manager to obtain and manage a TLS certificate from Let's Encrypt using the `letsencrypt-dns` ClusterIssuer.

## PushSecret Resource

Additionally, a `PushSecret` resource is defined to push the generated TLS certificate and its private key to 1Password. This allows other applications or systems to securely access the certificate material.

The `PushSecret` is configured as follows:

- **PushSecret Name:** `batlab-wildcard-cert`
- **Source Secret:** `certificate-batlab`
- **1Password Secret:** `batlab-wildcard-cert`

It pushes the following keys:

- `tls.crt` is pushed to the `certificate` property in 1Password.
- `tls.key` is pushed to the `privatekey` property in 1Password.

## Dependencies

This configuration relies on the following components:

- **[cert-manager](https://cert-manager.io/):** For automated certificate issuance and renewal.
- **[External Secrets](https://external-secrets.io/):** For pushing the generated certificate and key to 1Password.
