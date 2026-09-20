# ownCloud Infinite Scale (oCIS)

This directory contains the manifests for deploying [ownCloud Infinite Scale](https://owncloud.com/infinite-scale/) (oCIS).

oCIS is a self-hosted file sync-and-share platform (an ownCloud/Nextcloud-style personal cloud) shipped as a single Go binary that runs all of its microservices in one process.

## Image

This deployment uses the `owncloud/ocis` Docker image, running `ocis server` as a single container.

TLS is terminated upstream by the gateway/Cloudflare tunnel, so the container is configured with `PROXY_TLS=false` and `OCIS_INSECURE=true` to serve plain HTTP on port `9200`.

## Dependencies

This application has the following dependencies:

- **Storage:** It requires two PersistentVolumes provided by the `longhorn` storage class:
  - `ocis-config` (`/etc/ocis`): generated configuration, created on first start.
  - `ocis-data` (`/var/lib/ocis`): user files, metadata, and the identity (IDM) store.
- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to fetch the initial admin password from 1Password.

## Secrets

This application uses an ExternalSecret to create a secret named `ocis-secret`. This secret contains the following key, fetched from 1Password:

- **`idm_admin_password`**: The password for the built-in `admin` user, from the `ocis` item's `idm_admin_password` property. It is passed to the container as `IDM_ADMIN_PASSWORD`, which oCIS uses to (re)set the admin user's credentials on startup.

## Access

Exposed at `cloud.batlab.io` via both the external Gateway API route and a Cloudflare Tunnel `Ingress`.
