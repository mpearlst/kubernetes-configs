# Longhorn

This directory contains the manifests for deploying [Longhorn](https://longhorn.io/).

Longhorn is a lightweight, reliable, and easy-to-use distributed block storage system for Kubernetes.

## Helm Chart

This deployment uses the `longhorn` Helm chart from the [Longhorn chart repository](https://charts.longhorn.io/).

## StorageClass

This deployment defines a `StorageClass` named `longhorn-single` with the number of replicas set to 1. This is suitable for workloads where high availability is not a requirement.
