# ntfy

This directory contains the manifests for deploying [ntfy](https://ntfy.sh/).

ntfy is a simple, HTTP-based push notification service. It allows you to send notifications to your phone or desktop via a simple PUT/POST request.

## Image

This deployment uses the `binwiederhier/ntfy` Docker image.
## Dependencies

This application has the following dependencies:

- **Storage:** It requires two PersistentVolumes for user data and cache, provided by the `longhorn` storage class.
- **[External Secrets](https://external-secrets.io/):** It uses External Secrets to manage the `ntfy-config` secret.

## Secrets

This application uses an ExternalSecret to create a secret named `ntfy-config`. This secret contains the `server.yml` configuration file for ntfy.

- **Secret Name:** `ntfy-config`
- **Key:** `server.yml`

The `server.yml` file contains a `web_push_private_key` that is fetched from a 1Password secret named `ntfy` with the property `web_push_private_key`.
