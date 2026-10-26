# Cilium

This directory contains the manifests for deploying [Cilium](https://cilium.io/).

Cilium is an open source project that provides networking, observability, and security for container workloads. It is used as a CNI (Container Network Interface) plugin in this Kubernetes cluster.

## Helm Chart

This deployment uses the `cilium` Helm chart from the [Cilium Helm repository](https://helm.cilium.io/).

## Features

This deployment has the following features enabled:

- **Prometheus:** Agent, operator and Envoy metrics are enabled with ServiceMonitors. They had been turned off in `dac0dd9` during a worker2 OOM incident, but the real cause of that OOM turned out to be missing resource requests cluster-wide. Hubble metrics remain DISABLED (commented out in `application.yaml`); enabling them adds agent load, so watch agent memory against its 768Mi limit if you turn them on.
- **Hubble:** Disabled along with the metrics above - `hubble-relay` and `hubble-ui` are not running.
- **kube-proxy replacement:** Cilium replaces `kube-proxy` for service routing.
- **Gateway API:** Cilium is configured to work with the Kubernetes Gateway API.
- **L2 Announcements:** Advertises LoadBalancer IPs via ARP on the local L2 segment.
