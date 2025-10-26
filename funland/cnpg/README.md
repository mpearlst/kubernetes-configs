# Cloud Native PostgreSQL (CNPG)

This directory contains Kubernetes manifests and configurations related to [Cloud Native PostgreSQL](https://cloudnative-pg.io/).

Cloud Native PostgreSQL is an open-source Kubernetes operator that manages PostgreSQL databases within Kubernetes environments, covering their entire operational lifecycle.

## Directory Structure

- `cnpg-operator/`: Contains the manifests for deploying the Cloud Native PostgreSQL operator itself.
- `cnpg-clusters/`: Contains definitions for various PostgreSQL clusters managed by the operator, used by different applications.
- `cnpg-components/`: Contains additional components and configurations related to CNPG, such as backup configurations.
