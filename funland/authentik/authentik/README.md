# Authentik

This directory contains the manifests for deploying [Authentik](https://goauthentik.io/).

Authentik is an open-source Identity Provider (IdP) that you can self-host to provide Single Sign-On (SSO) for your applications.

## Helm Chart

This deployment uses the `authentik` Helm chart from the [Authentik chart repository](https://charts.goauthentik.io).

## Dependencies

This application has the following dependencies:

- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to fetch credentials from 1Password.
- **PostgreSQL:** It requires a PostgreSQL database. This deployment uses a database managed by the [Cloud Native PostgreSQL (CNPG) operator](https://cloudnative-pg.io/).
- **Redis:** It uses a Redis instance for caching, which is deployed as part of the Helm chart.

## Secrets

This application uses External Secrets to create a secret named `authentik-configsecrets` with the following keys:

- `bootstrap_password`: The password for the initial `akadmin` user.
- `bootstrap_token`: A token for initial setup and API access.
- `bootstrap_email`: The email address for the initial `akadmin` user.
- `secret_key`: A secret key for the application.
- `postgrespassword`: The password for the PostgreSQL database user.
- `postgresuser`: The username for the PostgreSQL database.

These keys are populated from two secrets in 1Password:

1.  **`authentik` secret:**
    - `bootstrap_password`
    - `bootstrap_token`
    - `bootstrap_email`
    - `secret_key`

2.  **`authentik-db-credentials` secret:**
    - `password` (maps to `postgrespassword`)
    - `username` (maps to `postgresuser`)
