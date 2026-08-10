# Recyclarr

This directory contains the manifests for deploying [Recyclarr](https://recyclarr.dev/), which syncs [TRaSH Guides](https://trash-guides.info/) quality profiles and custom formats into Radarr and Sonarr.

## Image

This deployment uses the `recyclarr/recyclarr` Docker image, pinned to major version `8` (receives feature/bugfix updates automatically within that major version, per the project's own recommendation).

## Access / SSO

None — Recyclarr has no web UI, so it gets no Service, HTTPRoute, or Authentik integration. It's a headless background sync job.

## How it runs

The container's default entrypoint runs in "Cron Mode": it stays alive and re-syncs on the schedule set by `CRON_SCHEDULE` (`@daily` here) — no separate Kubernetes CronJob is needed.

## Dependencies

- **Storage:** Longhorn PVC (`recyclarr-config`, 2Gi) for `/config` (cache/logs).
- **[External Secrets](https://external-secrets.io/):** Renders `/config/recyclarr.yml` from a template, reusing the **same** Radarr/Sonarr API keys those apps already consume (`radarr`/`sonarr` 1Password items, `api_key` property) — no separate key-extraction step.

## Secrets

`recyclarr-config` ExternalSecret templates `recyclarr.yml` with `radarr.radarr-main.api_key` and `sonarr.sonarr-main.api_key` filled in from 1Password.

## Manual setup

The shipped `recyclarr.yml` only defines the connection to Radarr/Sonarr (`base_url`/`api_key`/`quality_definition`) — it does not yet include `custom_formats`/`quality_profiles`. Add those per the [TRaSH Guides](https://trash-guides.info/) once you've decided which profiles you want synced.
