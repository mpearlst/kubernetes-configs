# Overseerr

This directory contains the manifests for deploying [Overseerr](https://overseerr.dev/) (via the [seerr](https://github.com/seerr-team/seerr) fork), a media request management UI for Radarr/Sonarr.

## Image

This deployment uses the `ghcr.io/seerr-team/seerr` Docker image.

## Access / SSO

Exposed as **`request.batlab.io`** on both the internal and external gateways. Unlike Prowlarr/Radarr/Sonarr, Overseerr is **not** routed through the Authentik proxy outpost — it has its own built-in authentication (local accounts / Plex / Jellyfin sign-in), so `http-route.yaml` points directly at this app's own Service with no SSO gate.

## Dependencies

- **Storage:** Longhorn PVC (`overseerr-config`, 5Gi) for `/app/config`.

## Manual setup (no declarative equivalent)

In Overseerr's setup wizard, connect it to Radarr and Sonarr using their API keys and in-cluster URLs (`http://radarr.media.svc.cluster.local:7878`, `http://sonarr.media.svc.cluster.local:8989`), and configure its own local authentication.
