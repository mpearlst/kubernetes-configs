# External Secrets Operator

This directory contains the manifests for deploying the [External Secrets Operator](https://external-secrets.io/).

The External Secrets Operator integrates Kubernetes with external secret management systems to securely inject secrets into your cluster.

## Helm Chart

This deployment uses the `external-secrets` Helm chart from the [External Secrets Helm repository](https://charts.external-secrets.io).

## Secrets

This application does not require any Kubernetes secrets for its own deployment. It is used to manage secrets for other applications.
