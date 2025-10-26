# Immich

This directory contains the manifests for deploying [Immich](https://immich.app/), a self-hosted photo and video backup solution.

## Helm Chart

This deployment uses the `immich` Helm chart from the [Immich chart repository](https://immich-app.github.io/immich-charts).

## Dependencies

This application has the following dependencies:

- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to fetch database credentials from 1Password.
- **PostgreSQL:** It requires a PostgreSQL database. This deployment uses a database managed by the [Cloud Native PostgreSQL (CNPG) operator](https://cloudnative-pg.io/).
- **Valkey:** It uses a Valkey instance for caching, which is deployed as part of the Helm chart.
- **NFS:** It requires a PersistentVolume for storing the photo and video library. This deployment uses an NFS-backed PersistentVolume.

## Secrets

This application requires a secret named `immich-secret` with the following keys:

- `DB_URL`
- `DB_HOSTNAME`
- `DB_PORT`
- `DB_USERNAME`
- `DB_PASSWORD`
- `DB_DATABASE_NAME`

This secret is created using External Secrets, which fetches the values from a 1Password secret named `immich-db-credentials` with the following properties:

- `username` (maps to `DB_USERNAME` and parts of `DB_URL`)
- `password` (maps to `DB_PASSWORD` and parts of `DB_URL`)
