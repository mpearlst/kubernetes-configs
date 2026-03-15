# Cilium L2 Announcements (Archived)

This directory contains the previous L2 announcement configuration that was replaced by BGP peering. These files are kept as a fallback option.

## What changed

The cluster was migrated from Cilium L2 announcements to BGP control plane peering with the UniFi router (commit `c43927a`).

| | L2 Announcements | BGP |
|---|---|---|
| **IP Pool** | `192.168.10.30` - `192.168.10.49` | `192.168.101.0/24` |
| **Internal Gateway** | `192.168.10.30` | `192.168.101.1` |
| **External Gateway** | `192.168.10.31` | `192.168.101.2` |
| **Mechanism** | ARP on local L2 segment | BGP peering with router (ASN 64512 ↔ 64514) |
| **Config file** | `l2-config.yaml` | `bgp-config.yaml` |

## Files in this directory

- `l2-config.yaml` - CiliumL2AnnouncementPolicy and CiliumLoadBalancerIPPool
- `gateway-internal.yaml` - Internal Gateway using L2 IP (`192.168.10.30`)
- `gateway-external.yaml` - External Gateway using L2 IP (`192.168.10.31`)

## Rollback procedure

To revert from BGP back to L2 announcements:

### 1. Update Cilium Helm values

In `funland/kube-system/cilium/application.yaml`, replace the BGP value:

```yaml
# Remove this:
bgpControlPlane:
  enabled: true

# Add this:
l2announcements:
  enabled: true
```

### 2. Replace the BGP config with L2 config

```sh
# Remove BGP configuration
rm funland/kube-system/cilium/bgp-config.yaml

# Copy L2 configuration into place
cp archive/cilium-l2/l2-config.yaml funland/kube-system/cilium/l2-config.yaml
```

### 3. Update Gateway IPs

```sh
cp archive/cilium-l2/gateway-internal.yaml funland/gateway/gateway/internal-gateway.yaml
cp archive/cilium-l2/gateway-external.yaml funland/gateway/gateway/external-gateway.yaml
```

### 4. Update any services using static IPs from the BGP pool

Services with `io.cilium/lb-ipam-ips` annotations in the `192.168.101.x` range need to be updated to use IPs in the `192.168.10.30-49` range. Currently these are:

- `funland/adguard/adguard-home/service.yaml` → change `192.168.101.5` to an IP in `192.168.10.30-49`
- `funland/unifi-log-insight/unifi-log-insight/service.yaml` → change `192.168.101.3` to an IP in `192.168.10.30-49`
- `funland/openspeedtest/openspeedtest/service.yaml` → change `192.168.101.6` to an IP in `192.168.10.30-49`

### 5. Remove BGP configuration from UniFi router

Disable BGP peering on the UniFi router since it will no longer be needed.

### 6. Update DNS records

Update any DNS records pointing to `192.168.101.x` addresses back to `192.168.10.x` addresses.

### 7. Commit and push

ArgoCD will automatically sync the changes once pushed.
