# External Secrets

This directory contains Kubernetes manifests and configurations related to the [External Secrets Operator](https://external-secrets.io/).

The External Secrets Operator integrates Kubernetes with external secret management systems like 1Password, AWS Secrets Manager, HashiCorp Vault, etc., to securely inject secrets into your cluster.

## Directory Structure

- `external-secrets/`: Contains the manifests for deploying the External Secrets Operator itself.
- `cluster-secret-store/`: Contains definitions for ClusterSecretStore resources, which define how to access external secret management systems (e.g., 1Password).
- `1password-connect/`: Contains configurations specific to connecting with 1Password, likely including the 1Password Connect server deployment.
