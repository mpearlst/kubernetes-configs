# Kubernetes Gateway API Configuration

This directory contains Kubernetes Gateway API resources for managing ingress and egress traffic within the cluster.

## Resources

- `external-gateway.yaml`: Defines a Gateway resource configured to handle external traffic coming into the cluster.
- `internal-gateway.yaml`: Defines a Gateway resource configured to handle internal traffic within the cluster.
- `http-route.yaml`: Redirects plain HTTP (port 80) on both Gateways to HTTPS with a 301. Per-app HTTPRoutes live alongside each app and attach to the `https` listener.

## Dependencies

Both Gateways use the `cilium` GatewayClass provided by [Cilium](https://cilium.io/)'s Gateway API support, with fixed LoadBalancer IPs (`internal`: `192.168.10.30`, `external`: `192.168.10.31`).

### Gateway API CRDs

The Gateway API CRDs are managed separately in `funland/gateway-api/gateway-api-crds/` and must be installed before these resources can be applied.

The pinned release is the URL in `funland/gateway-api/gateway-api-crds/crds/kustomization.yaml` (Renovate opens PRs for new releases; they are never auto-merged).

The CRD installation uses an App of Apps pattern with **Server-Side Apply** (`ServerSideApply=true`) enabled. This is required because the Gateway API CRD manifests are large and can exceed the 1MB annotation limit used by client-side apply.

## Secrets

The `external-gateway.yaml` and `internal-gateway.yaml` resources both require a TLS secret for HTTPS traffic. The details are as follows:

- **Secret Name:** `batlab`
- **Namespace:** `certificate`

This secret should contain the TLS certificate and private key for `*.batlab.io`.
