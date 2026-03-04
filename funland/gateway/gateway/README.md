# Kubernetes Gateway API Configuration

This directory contains Kubernetes Gateway API resources for managing ingress and egress traffic within the cluster.

## Resources

- `external-gateway.yaml`: Defines a Gateway resource configured to handle external traffic coming into the cluster.
- `internal-gateway.yaml`: Defines a Gateway resource configured to handle internal traffic within the cluster.
- `http-route.yaml`: Defines HTTPRoute resources that specify how HTTP traffic is routed from the Gateways to various services within the cluster.

## Dependencies

This configuration relies on a Kubernetes Gateway API implementation being deployed in the cluster, such as [Cilium](https://cilium.io/) or Istio.

### Gateway API CRDs

The Gateway API CRDs are managed separately in `funland/gateway-api/gateway-api-crds/` and must be installed before these resources can be applied.

**Current version**: v1.5.0

The CRD installation uses an App of Apps pattern with **Server-Side Apply** (`ServerSideApply=true`) enabled. This is required because the Gateway API CRD manifests are large and can exceed the 1MB annotation limit used by client-side apply.

## Secrets

The `external-gateway.yaml` and `internal-gateway.yaml` resources both require a TLS secret for HTTPS traffic. The details are as follows:

- **Secret Name:** `batlab`
- **Namespace:** `certificate`

This secret should contain the TLS certificate and private key for `*.batlab.io`.
