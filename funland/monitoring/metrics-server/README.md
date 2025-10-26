# Kubernetes Metrics Server

This directory contains the manifests for deploying the [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server).

The Kubernetes Metrics Server is a cluster-wide aggregator of resource usage data. It's a core component for enabling features like the Horizontal Pod Autoscaler (HPA).

## Helm Chart

This deployment uses the `metrics-server` Helm chart from the [Kubernetes SIGs repository](https://kubernetes-sigs.github.io/metrics-server/).

