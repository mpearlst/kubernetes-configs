# Archive

Retired manifests kept for reference. Nothing here is deployed: the ArgoCD
ApplicationSet only watches `funland/*/*`, and Renovate ignores `archive/**`.

To bring an app back, move its directory back under `funland/<namespace>/<app>/`,
update image and chart versions (they have not been maintained here), and
re-create any 1Password items its `ExternalSecret`s expect.
