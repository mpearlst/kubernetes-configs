# Cloud Native PostgreSQL (CNPG)

This directory contains Kubernetes manifests and configurations related to [Cloud Native PostgreSQL](https://cloudnative-pg.io/).

Cloud Native PostgreSQL is an open-source Kubernetes operator that manages PostgreSQL databases within Kubernetes environments, covering their entire operational lifecycle.

## Directory Structure

- `cnpg-operator/`: Contains the manifests for deploying the Cloud Native PostgreSQL operator itself.
- `cnpg-clusters/`: Contains definitions for various PostgreSQL clusters managed by the operator, used by different applications.
- `cnpg-components/`: Contains the ExternalSecret for the Backblaze B2 backup credentials.

Each subdirectory is deployed as its own ArgoCD Application by the ApplicationSet. All database clusters live in the `postgres` namespace (declared in `cnpg-clusters/namespace.yaml`).

See [QUICKSTART.md](QUICKSTART.md) for day-to-day backup operations and [README-BACKUPS.md](README-BACKUPS.md) for the full backup and restore reference.
