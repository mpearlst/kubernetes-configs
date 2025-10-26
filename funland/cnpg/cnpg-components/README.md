# Cloud Native PostgreSQL Components

This directory contains additional components and configurations related to [Cloud Native PostgreSQL](https://cloudnative-pg.io/), primarily focusing on backup configurations.

## Contents

- `b2-backup-credentials.yaml`: Contains an ExternalSecret definition for fetching Backblaze B2 backup credentials.
- `b2-backup/`: This subdirectory contains further configurations related to Backblaze B2 backups, such as `backup-config.yaml`.

## Secrets

The `b2-backup-credentials.yaml` file defines an `ExternalSecret` that creates a secret named `b2-backup-credentials` in the `postgres` namespace. This secret is used for authenticating to Backblaze B2 for backups.

- **Secret Name:** `b2-backup-credentials`
- **Keys:**
  - `ACCESS_KEY_ID`
  - `ACCESS_SECRET_KEY`

These keys are populated from a 1Password secret named `b2-backup-credentials` with the following properties:

- `keyId` (maps to `ACCESS_KEY_ID`)
- `applicationKey` (maps to `ACCESS_SECRET_KEY`)
