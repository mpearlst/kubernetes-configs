# Vikunja

This directory contains the manifests for deploying [Vikunja](https://vikunja.io/).

Vikunja is an open-source, self-hostable to-do and task management application designed to help individuals and teams organize their work and projects.

## Image

This deployment uses the `vikunja/vikunja` Docker image.
## Dependencies

This application has the following dependencies:

- **PostgreSQL:** It requires a PostgreSQL database. This deployment uses a database managed by the [Cloud Native PostgreSQL (CNPG) operator](https://cloudnative-pg.io/).
- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to fetch various credentials (JWT secret, PostgreSQL credentials, SSO client ID/secret) from 1Password.
- **[Authentik](https://goauthentik.io/):** It is configured to use Authentik for OpenID Connect (OIDC) based authentication.

## Secrets

This application uses two ExternalSecrets to manage its credentials:

1.  **`vikunja-configsecrets`**: This secret contains:
    - `jwtsecret`: From the `vikunja` secret in 1Password.
    - `postgrespassword`: From the `vikunja-db-credentials` secret in 1Password.
    - `postgresuser`: From the `vikunja-db-credentials` secret in 1Password.

2.  **`vikunja-sso`**: This secret contains the `config.yaml` file for OIDC configuration and includes:
    - `client_id`: From the `vikunja-sso` secret in 1Password (property: `vikunja_client_id`).
    - `client_secret`: From the `vikunja-sso` secret in 1Password (property: `vikunja_client_secret`).
