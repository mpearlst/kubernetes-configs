# Synapse

This directory contains the manifests for deploying [Synapse](https://matrix.org/docs/projects/server/synapse).

Synapse is an open-source Matrix homeserver, which is the reference implementation of the Matrix open standard for decentralized, secure, and interoperable real-time communication.

## Image

This deployment uses the `ghcr.io/element-hq/synapse` Docker image.
## Dependencies

This application has the following dependencies:

- **Storage:** It requires a PersistentVolume for storing data, provided by the `longhorn` storage class.
- **PostgreSQL:** It requires a PostgreSQL database. This deployment uses a database managed by the [Cloud Native PostgreSQL (CNPG) operator](https://cloudnative-pg.io/).
- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to manage the `synapse-config` secret.

## Secrets

This application uses an ExternalSecret to create a secret named `synapse-config`. This secret contains the `homeserver.yaml` configuration file for Synapse.

- **Secret Name:** `synapse-config`
- **Key:** `homeserver.yaml`

The `homeserver.yaml` file contains several secrets that are fetched from 1Password:

- **From the `synapse-db-credentials` secret:**
  - `password` (as `synapse_db_pass`)
  - `username` (as `synapse_db_user`)

- **From the `synapse` secret:**
  - `synapse-registration-secret`
  - `synapse-macaroon-secret`
  - `synapse-form-secret`

- **From the `synapse-oidc` secret:**
  - `client_id` (as `oidc_client_id`)
  - `client_secret` (as `oidc_client_secret`)
