# AdGuard Home

This directory contains the Kubernetes manifests for deploying [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome).

AdGuard Home is a network-wide software for blocking ads and tracking. After you set it up, it'll cover ALL your home devices, and you don't need any client-side software for that.

## Image

This deployment uses the `adguard/adguardhome` Docker image.

## Dependencies

This application has the following dependencies:

- **Storage:** It requires two PersistentVolumes for storing configuration and data, provided by the `longhorn` storage class.

## Networking

- **DNS:** `adguard-home-dns` is a `LoadBalancer` Service on `192.168.10.35` (53/tcp, 53/udp, 853/tcp DNS-over-TLS), announced by Cilium L2 with DSR forwarding.
- **Web UI:** `adguard.batlab.io` via the internal Gateway (`adguard-home-http`, port 80).
- The first-run setup wizard listens on port 3000; the commented-out `adguard-home-setup` Service in `service.yaml` can be re-enabled to reach it on a fresh install.
