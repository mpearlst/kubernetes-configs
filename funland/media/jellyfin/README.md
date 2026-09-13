# Jellyfin

This directory contains the manifests for deploying [Jellyfin](https://jellyfin.org/).

Jellyfin is a free software media system that puts you in control of managing and streaming your media.

## Image

This deployment uses the `jellyfin/jellyfin` Docker image.

## Dependencies

This application has the following dependencies:

- **Storage:**
  - It requires a PersistentVolume for storing configuration, provided by the `longhorn` storage class.
  - It connects to an NFS share for the media library.
  - Downloaded artwork/metadata (`/config/metadata`) lives on a separate NFS share (`/mnt/tank/jellyfin/metadata`) rather than the Longhorn volume, since it's the bulk of the config volume's size and is safely re-fetchable if lost.
