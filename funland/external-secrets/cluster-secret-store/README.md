# ClusterSecretStore Configuration

This directory contains the definition for a `ClusterSecretStore` resource, which configures how the [External Secrets Operator](https://external-secrets.io/) connects to an external secret management system.

## 1Password ClusterSecretStore

The `cluster-secret-store.yaml` file defines a `ClusterSecretStore` named `1password`. This resource specifies the connection details for a 1Password Connect server, allowing the External Secrets Operator to retrieve secrets from 1Password vaults.

## Dependencies

This configuration relies on the following components:

- **[External Secrets Operator](https://external-secrets.io/):** The operator that utilizes this `ClusterSecretStore` to fetch secrets.
- **1Password Connect Server:** An instance of the 1Password Connect server must be running and accessible within the cluster.

## Secrets

This `ClusterSecretStore` requires a secret for authentication with the 1Password Connect server. The details are as follows:

- **Secret Name:** `1passwordconnect`
- **Namespace:** `external-secrets`
- **Key:** `token`
