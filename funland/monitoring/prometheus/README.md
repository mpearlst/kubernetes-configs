# Prometheus

This directory contains the manifests for deploying [Prometheus](https://prometheus.io/).

Prometheus is an open-source systems monitoring and alerting toolkit. It collects and stores metrics as time series data, which includes a timestamp and optional key-value pairs called labels.

## Helm Chart

This deployment uses the `kube-prometheus-stack` Helm chart from the [Prometheus Community Helm Charts repository](https://prometheus-community.github.io/helm-charts).

## Alerting

Alertmanager is enabled (with a small Longhorn-backed PVC for its state) and configured with a single `ntfy` receiver — everything except the `Watchdog` heartbeat alert routes there. The chart's default rules already include `KubeNodeNotReady` and `KubeletDown`, which cover node/kubelet-hang failures.

Alertmanager's webhook receiver posts its own JSON schema, which ntfy can't render directly, so the receiver points at `funland/monitoring/alertmanager-ntfy-bridge` (see that directory's README for the one-time ntfy token setup) rather than ntfy itself.

This was added after a 2026-08-16 incident: `talos-worker1` went `NotReady` for ~26 hours with no alerting in place to catch it. See `funland/longhorn-system/longhorn/README.md` for the storage-side half of the same fix.
