# Prometheus

This directory contains the manifests for deploying [Prometheus](https://prometheus.io/).

Prometheus is an open-source systems monitoring and alerting toolkit. It collects and stores metrics as time series data, which includes a timestamp and optional key-value pairs called labels.

## Helm Chart

This deployment uses the `kube-prometheus-stack` Helm chart from the [Prometheus Community Helm Charts repository](https://prometheus-community.github.io/helm-charts).

## Alerting

Alertmanager is enabled (with a small Longhorn-backed PVC for its state). Alerts route to an `ntfy` receiver; the `Watchdog` heartbeat goes to an external dead-man's-switch instead (see below). The chart's default rules already include `KubeNodeNotReady` and `KubeletDown`, which cover node/kubelet-hang failures.

Alertmanager's webhook receiver posts its own JSON schema, which ntfy can't render directly, so the receiver points at `funland/monitoring/alertmanager-ntfy-bridge` (see that directory's README for the one-time ntfy token setup) rather than ntfy itself.

This was added after a 2026-08-16 incident: `talos-worker1` went `NotReady` for ~26 hours with no alerting in place to catch it. See `funland/longhorn-system/longhorn/README.md` for the storage-side half of the same fix.

## 2026-08-23: node flapping and the alert flood

Nodes kept dropping out and the ntfy stream became unusable. Investigation found one root cause plus several noise amplifiers.

**Root cause — the scheduler's view of node memory was fiction.** Most chart-deployed workloads declared no memory requests, and a pod with no request counts as zero to the scheduler:

| node | pods | mem requested | mem actually used | 7d kernel OOM kills |
|---|---|---|---|---|
| talos-worker1 | 17 | 208Mi (2%) | ~4.0Gi (54%) | 0 |
| talos-worker2 | 53 | 586Mi (7%) | ~5.5Gi (71%) | 153 |
| talos-worker3 | 21 | 5.03Gi (69%) | ~4.4Gi (58%) | 4 |

worker2 advertised itself as 93% free while actually 71% full, so the scheduler kept packing pods onto it until the kernel OOM killer fired. worker3 was the only stable worker precisely because its workloads (`funland/media/*`) set explicit requests, so the scheduler routed around it. These were real VM reboots — `node_boot_time_seconds` changed and all node conditions reset together.

Fix: requests (and, where safe, memory limits) were set across every chart-deployed workload in this repo, plus kubelet `systemReserved`/`kubeReserved`/`evictionHard` in `funland/talos/talconfig.yaml`. The kubelet reservations are load-bearing, not belt-and-braces: Longhorn creates ~25 of its pods outside Helm and exposes no memory requests for them, so the scheduler will always under-count Longhorn.

**Noise amplifiers fixed at the same time:**

- `KubeProxyDown`, `KubeSchedulerInstanceUnreachable`, `KubeControllerManagerInstanceUnreachable` and two `TargetDown` fired permanently — kube-proxy is disabled by design (Cilium replaces it) and Talos binds the scheduler/controller-manager to localhost. Those chart components are now `enabled: false`.
- `group_by` was `['namespace']` only, so one node failure fanned out into a separate ntfy push per affected namespace. Now `['alertname', 'namespace']`, with a dedicated node-down route and an inhibit rule so `KubeNodeNotReady` suppresses the downstream pod-level noise.
- The cloudflared connector had been in `ImagePullBackOff` since 2026-08-12, keeping `KubeDeploymentRolloutStuck` and `KubePodNotReady` permanently lit. Chart 0.1.0 changed the connector image to a fork with its own tag scheme while Renovate was still writing upstream cloudflared tags into it. See `funland/cloudflare/cloudflare/application.yaml`.
- Longhorn's `PrometheusRule` was labelled `prometheus: longhorn` but the Prometheus CR selects `release: prometheus-application`, so those five alerts had never loaded. Separately, Longhorn 1.12.x NetworkPolicies were dropping the metrics scrape entirely — all three `longhorn-backend` targets were down and no `longhorn_*` metrics existed. Both fixed in `funland/longhorn-system/longhorn/`.

**Resilience changes:** Prometheus, Alertmanager, kube-state-metrics and the ntfy bridge now carry soft anti-affinity so the alert path is not co-located on one node — previously Prometheus, kube-state-metrics and the bridge all ran on worker2, the node that kept dying. Prometheus also gained a Longhorn PVC and 30d retention; it was on an `emptyDir`, so every restart destroyed the evidence and re-fired every alert at once.

`metrics-server` was also restored (`funland/kube-system/metrics-server/`) — `kubectl top` is the fastest way to spot a requests-vs-usage gap like this one.

## Dead-man's-switch (Watchdog)

The entire alert path is in-cluster and single-replica (Prometheus → Alertmanager → bridge → ntfy), so a failure of the path itself produces silence rather than a notification. `Watchdog` fires continuously by design and is posted to an **external** heartbeat service, which alarms when the pings stop.

The ping URL is a capability secret and this repository is public, so it lives in 1Password and is mounted via `external-secret-watchdog.yaml`.

**Bootstrap — do this before syncing this app**, otherwise the Alertmanager pod stays in `ContainerCreating` waiting for the secret:

1. Create a check at an external heartbeat service (healthchecks.io, Dead Man's Snitch, or a self-hosted endpoint **outside** this cluster). Period ~10m, grace ~20m — `Watchdog` repeats every 5m.
2. Create a 1Password item `alertmanager-watchdog` in the same vault as the other cluster secrets, with a property `url` holding the ping URL.
3. Sync, then confirm the heartbeat service shows "up".

To verify it actually works, scale the ntfy bridge to 0 and confirm the external service alarms.
