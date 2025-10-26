# Vaultwarden

This directory contains the manifests for deploying [Vaultwarden](https://vaultwarden.github.io/).

Vaultwarden is an open-source, self-hosted password manager server compatible with Bitwarden clients. It's a lightweight alternative written in Rust.

## Image

This deployment uses the `vaultwarden/server` Docker image.
## Dependencies

This application has the following dependencies:

- **PostgreSQL:** It requires a PostgreSQL database. This deployment uses a database managed by the [Cloud Native PostgreSQL (CNPG) operator](https://cloudnative-pg.io/).
- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to fetch various credentials (admin token, push keys, SMTP settings, database URL) from 1Password.
- **SMTP Server:** An external SMTP server is required for email notifications (e.g., for new user sign-ups, password resets).

## Secrets

This application uses an ExternalSecret to create a secret named `vaultwarden-secret`. This secret contains the following keys, fetched from 1Password:

- **`database-url`**: Constructed from the `vaultwarden-db-credentials` secret.
- **`vaultwarden_admin_token`**: From the `vaultwarden` secret.
- **`push_installation_id`**: From the `vaultwarden` secret.
- **`push_installation_key`**: From the `vaultwarden` secret.
- **`smtp_address`**: From the `smtp` secret.
- **`smtp_user`**: From the `smtp` secret.
- **`smtp_password`**: From the `smtp` secret.
