# n8n

This directory contains the manifests for deploying [n8n](https://n8n.io/).

n8n is a free and open-source workflow automation tool. It helps you to connect different applications and automate tasks without writing code.

## Image

This deployment uses the `n8nio/n8n` Docker image.
## Dependencies

This application has the following dependencies:

- **PostgreSQL:** It requires a PostgreSQL database. This deployment uses a database managed by the [Cloud Native PostgreSQL (CNPG) operator](https://cloudnative-pg.io/).
- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to fetch database credentials from 1Password.

## Secrets

This application requires a secret named `n8n-secrets` with the following key:

- `dbpass`

This secret is created using External Secrets, which fetches the value from a 1Password secret named `n8n-db-credentials` with the property `password`.
