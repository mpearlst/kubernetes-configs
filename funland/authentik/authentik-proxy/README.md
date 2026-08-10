# Authentik Proxy Outpost

This directory contains a shared [Authentik](https://goauthentik.io/) **Proxy Provider outpost** that provides SSO for apps with no native OIDC support (Prowlarr, Radarr, Sonarr — see `funland/media/`).

## Why this exists

Cilium's Gateway API `HTTPRoute` has no forward-auth filter, so there's no gateway-level way to gate traffic on an Authentik login. Instead, a single outpost Deployment (`ghcr.io/goauthentik/proxy`) sits in the request path and does the job itself: each protected app's `HTTPRoute` points here instead of at its own Service; the outpost checks the request against Authentik, and only forwards it on to the real app (its `internal_host`, declared in the Proxy Provider config below) once authenticated. One outpost pod serves all three apps simultaneously via Host-header routing — no per-app outpost needed.

## Files

- `deployment.yaml` — the outpost Deployment + Service (port 9000). `AUTHENTIK_HOST` points at the same internal Service (`authentik-application-server`, port 80) that `funland/authentik/authentik/http-route.yaml` already uses; `AUTHENTIK_HOST_BROWSER` is the public `auth.batlab.io` hostname, used for browser-facing redirects.
- `external-secret.yaml` — pulls the outpost's API token from the `authentik-proxy-outpost` 1Password item (`token` property) into `authentik-proxy-secret`.
- `reference-grant.yaml` — permits `HTTPRoute`s in the `media` namespace to reference this folder's `authentik-proxy` Service (cross-namespace Gateway API backendRefs require this).
- `blueprint-configmap.yaml` — an [Authentik Blueprint](https://docs.goauthentik.io/customize/blueprints/) declaring the `media-admins` group, one Proxy Provider + Application per protected app, a policy binding restricting each application to `media-admins`, and the outpost itself (listing all three providers). Mounted into the authentik server/worker via the chart's native `blueprints.configMaps` value (see `funland/authentik/authentik/application.yaml`).

## Bootstrap (one manual step)

1. Create a 1Password item `authentik-proxy-outpost` with a placeholder `token` property before first deploy, so the ExternalSecret has something to sync.
2. After the blueprint applies (Admin UI → **System → Blueprints**, `media-proxy.yaml` should show `successful`), go to **Directory → Tokens**, find the auto-generated token for the `media-proxy-outpost` outpost's service account, and copy its value into the `authentik-proxy-outpost` 1Password item's `token` property.
3. Let the ExternalSecret refresh (≤5m) and restart the outpost: `kubectl -n authentik rollout restart deploy/authentik-proxy`.

There's no fully-declarative way around this step without deeper verification of Authentik's blueprint `!Env` token support — this manual copy-paste is the guaranteed-safe path.

## Verification

- `kubectl -n authentik logs deploy/authentik-proxy` — should show no `401`/invalid-token errors once the real token is in place.
- `curl -sI https://prowlarr.batlab.io/ --resolve prowlarr.batlab.io:443:192.168.10.30` should redirect to `auth.batlab.io`, not return Prowlarr directly.
