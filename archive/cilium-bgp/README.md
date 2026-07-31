# Cilium BGP Control Plane (Archived)

This directory contains the BGP peering configuration that was replaced by
L2 announcements. These files are kept as a fallback option.

## What changed

The cluster was reverted from Cilium BGP control plane peering back to L2
announcements so devices on the LAN could be migrated to new IP addresses
without needing continuous BGP routing from the UniFi router.

| | BGP | L2 Announcements |
|---|---|---|
| **IP Pool** | `192.168.101.0/24` | `192.168.10.30` - `192.168.10.49` |
| **Internal Gateway** | `192.168.101.1` | `192.168.10.30` |
| **External Gateway** | `192.168.101.2` | `192.168.10.31` |
| **Mechanism** | BGP peering with router (ASN 64512 ↔ 64514) | ARP on local L2 segment |
| **Config file** | `bgp-config.yaml` | `l2-config.yaml` |

## Files in this directory

- `bgp-config.yaml` - CiliumBGPClusterConfig, CiliumBGPPeerConfig, CiliumBGPAdvertisement, and CiliumLoadBalancerIPPool
- `gateway-internal.yaml` - Internal Gateway using the BGP IP (`192.168.101.1`)
- `gateway-external.yaml` - External Gateway using the BGP IP (`192.168.101.2`)

## Roll-forward procedure

To move from L2 announcements back to BGP:

### 1. Update Cilium Helm values

In `funland/kube-system/cilium/application.yaml`, replace the L2 value:

```yaml
# Remove this:
l2announcements:
  enabled: true

# Add this:
bgpControlPlane:
  enabled: true
```

### 2. Replace the L2 config with the BGP config

```sh
# Remove L2 configuration
rm funland/kube-system/cilium/l2-config.yaml

# Copy BGP configuration into place
cp archive/cilium-bgp/bgp-config.yaml funland/kube-system/cilium/bgp-config.yaml
```

### 3. Update Gateway IPs

```sh
cp archive/cilium-bgp/gateway-internal.yaml funland/gateway/gateway/internal-gateway.yaml
cp archive/cilium-bgp/gateway-external.yaml funland/gateway/gateway/external-gateway.yaml
```

### 4. Update any services using static IPs from the L2 pool

Services with `loadBalancerIP` in the `192.168.10.30-49` range need to be
updated to use IPs in the `192.168.101.x` range. Currently these are:

- `funland/adguard/adguard-home/service.yaml` → change `192.168.10.32` to an IP in `192.168.101.0/24`
- `funland/openspeedtest/openspeedtest/service.yaml` → change `192.168.10.33` to an IP in `192.168.101.0/24`

### 5. Re-establish BGP peering on the UniFi router

Re-enable BGP peering (local ASN 64512, peer ASN 64514) on the UniFi router.

### 6. Update DNS records

Update any DNS records pointing to `192.168.10.x` addresses back to
`192.168.101.x` addresses.

### 7. Commit and push

ArgoCD will automatically sync the changes once pushed.
