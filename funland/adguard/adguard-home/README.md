# AdGuard Home

This directory contains the Kubernetes manifests for deploying [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome).

AdGuard Home is a network-wide software for blocking ads and tracking. After you set it up, it'll cover ALL your home devices, and you don't need any client-side software for that.

## Image

This deployment uses the `adguard/adguardhome` Docker image.

## Dependencies

This application has the following dependencies:

- **Storage:** It requires two PersistentVolumes for storing configuration and data, provided by the `longhorn` storage class.
