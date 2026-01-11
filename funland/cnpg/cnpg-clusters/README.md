# Cloud Native PostgreSQL Clusters

This directory contains the definitions for various PostgreSQL clusters managed by the [Cloud Native PostgreSQL Operator](https://cloudnative-pg.io/). Each cluster is typically dedicated to a specific application.

## Defined Clusters

- `authentik-cnpg-cluster.yaml`: PostgreSQL cluster for Authentik.
- `immich-cnpg-cluster.yaml`: PostgreSQL cluster for Immich.
- `n8n-cnpg-cluster.yaml`: PostgreSQL cluster for n8n.
- `vaultwarden-cnpg-cluster.yaml`: PostgreSQL cluster for Vaultwarden.
- `vikunja-cnpg-cluster.yaml`: PostgreSQL cluster for Vikunja.

## Scheduled Backups

The `scheduled-backups.yaml` file defines scheduled backup configurations for these PostgreSQL clusters.
