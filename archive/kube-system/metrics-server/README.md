# Kubernetes Metrics Server

This directory contains the manifests for deploying the [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server).

The Kubernetes Metrics Server is a cluster-wide aggregator of resource usage data. It's a core component for enabling features like the Horizontal Pod Autoscaler (HPA).

## Helm Chart

This deployment uses the `metrics-server` Helm chart from the [Kubernetes SIGs repository](https://kubernetes-sigs.github.io/metrics-server/).


## History

Archived in `dac0dd9` (2026-08-17) while chasing a `talos-worker2` OOM — the commit message itself noted it was "not the source of the OOM".

Restored 2026-08-23. The actual cause of that OOM, and of the repeated node `NotReady` flaps, was that most chart-deployed workloads declared no memory requests, so the scheduler saw worker2 as 7% full while it was really 71% full and kept packing pods onto it until the kernel OOM killer fired. `kubectl top nodes` / `kubectl top pods` is the fastest way to see that gap — without metrics-server the only route was raw `container_memory_working_set_bytes` queries against Prometheus.

It costs ~50Mi and now declares its own requests and limits, so it cannot repeat the accounting problem it was archived for. See `funland/monitoring/prometheus/README.md` for the full incident write-up.
