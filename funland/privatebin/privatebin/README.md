# PrivateBin

This directory contains the manifests for deploying [PrivateBin](https://privatebin.info/).

PrivateBin is a minimalist, open-source pastebin where the server has zero knowledge of pasted data. Data is encrypted/decrypted in the browser using 256-bit AES.

## Image

This deployment uses the `ghcr.io/privatebin/nginx-fpm-alpine` Docker image.
## Dependencies

This application has the following dependencies:

- **Storage:** It requires a PersistentVolume for storing data, provided by the `longhorn` storage class.
