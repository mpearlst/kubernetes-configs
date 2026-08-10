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
- **ConfigMap:** `recyclarr-yml` holds `/config/recyclarr.yml` in git — quality profiles and custom formats live here in plain sight, and secrets are never inlined into it. Values that need to stay secret use Recyclarr's [`!secret`](https://recyclarr.dev/wiki/yaml/secrets-reference/) YAML tag (e.g. `api_key: !secret radarr_apikey`), which Recyclarr resolves against `/config/secrets.yml` at runtime.
- **[External Secrets](https://external-secrets.io/):** Renders `/config/secrets.yml`, reusing the **same** Radarr/Sonarr API keys those apps already consume (`radarr`/`sonarr` 1Password items, `api_key` property) — no separate key-extraction step.

## Secrets

`recyclarr-secret` ExternalSecret templates `secrets.yml` with `radarr_url`/`radarr_apikey`/`sonarr_url`/`sonarr_apikey` filled in from the existing `radarr`/`sonarr` 1Password items, mounted read-only at `/config/secrets.yml`. `recyclarr.yml` references these via `!secret <key>` — it holds no secret material itself, so it's safe to edit and diff in git.

## Manual setup

Edit the `recyclarr-yml` ConfigMap's `recyclarr.yml` to add/adjust `custom_formats`/`quality_profiles` per the [TRaSH Guides](https://trash-guides.info/) — no 1Password changes needed unless you're changing which Radarr/Sonarr instances it talks to.
