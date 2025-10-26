# Cilium

This directory contains the manifests for deploying [Cilium](https://cilium.io/).

Cilium is an open source project that provides networking, observability, and security for container workloads. It is used as a CNI (Container Network Interface) plugin in this Kubernetes cluster.

## Helm Chart

This deployment uses the `cilium` Helm chart from the [Cilium Helm repository](https://helm.cilium.io/).

## Features

This deployment has the following features enabled:

- **Prometheus:** Metrics are exposed for Prometheus to scrape.
- **Hubble:** Hubble is enabled for network observability.
- **kube-proxy replacement:** Cilium replaces `kube-proxy` for service routing.
- **Gateway API:** Cilium is configured to work with the Kubernetes Gateway API.
- **L2 Announcements:** Allows services to be exposed on the local network.
