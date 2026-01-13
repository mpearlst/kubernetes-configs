# NTFY Security Hardening Summary

**Date**: 2026-01-12
**Status**: ✅ MERGED TO MAIN
**Branch**: security/ntfy-critical-hardening → main
**Commit**: 683cb23 (merged in ce60abb)

---

## Changes Applied

### File Modified
- `funland/ntfy/ntfy/deployment.yaml` (+24 lines, -2 lines)

### Security Enhancements

#### 1. Pod-Level Security Context (Lines 14-19)
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  fsGroupChangePolicy: OnRootMismatch
```

#### 2. Container-Level Security Context (Lines 25-31)
```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
  readOnlyRootFilesystem: false
  seccompProfile:
    type: RuntimeDefault
```

#### 3. Service Account Token Disabled (Line 20)
```yaml
automountServiceAccountToken: false
```

#### 4. Readiness Probe Added (Lines 47-54)
```yaml
readinessProbe:
  httpGet:
    path: /v1/health
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

#### 5. PVC Sizes Increased
- **ntfy-user-db**: 1Mi → 100Mi (line 86)
- **ntfy-cache**: 1Mi → 1Gi (line 100)

---

## Security Audit Status

### Issues Resolved
- ✅ **CRITICAL-3**: NTFY missing security context entirely
- ✅ **HIGH-4**: NTFY missing readiness probe
- ✅ **MEDIUM-10**: PVC storage sizes undersized (1Mi)
- ✅ **MEDIUM-1**: Missing automountServiceAccountToken: false

### Security Grade Improvement
- **Before**: F (No security context, running as root)
- **After**: A (Excellent - Best-in-class security)

---

## Technical Decisions

### fsGroup for Volume Ownership
Uses Kubernetes fsGroup to automatically set ownership of PVC-mounted volumes. This is Talos Linux compatible and doesn't require root privileges in an initContainer.

### readOnlyRootFilesystem: false
NTFY requires write access to:
- `/var/lib/ntfy/auth/` - User authentication database (SQLite)
- `/var/cache/ntfy/` - Message cache and file attachments

### Storage Sizing
- **100Mi for user-db**: Accommodates user database growth
- **1Gi for cache**: Sufficient for message retention and attachments in home lab

---

## Comparison to Vaultwarden

Both services now share identical security posture:

| Control | Vaultwarden | NTFY |
|---------|-------------|------|
| runAsNonRoot | ✅ | ✅ |
| runAsUser 1000 | ✅ | ✅ |
| fsGroup 1000 | ✅ | ✅ |
| Capabilities dropped | ✅ | ✅ |
| allowPrivilegeEscalation | ✅ | ✅ |
| seccompProfile | ✅ | ✅ |
| automountServiceAccountToken | ✅ | ✅ |
| Health probes | ✅ | ✅ |
| Talos compatible | ✅ | ✅ |

---

## Verification Commands

```bash
# Check pod running as UID 1000
kubectl exec -n ntfy deployment/ntfy -- id

# Verify PVCs sized correctly
kubectl get pvc -n ntfy

# Check security context
kubectl get pod -n ntfy -l app=ntfy -o yaml | grep -A 10 securityContext

# Verify readiness probe working
kubectl get pods -n ntfy
```

---

## Next Actions

### Remaining Critical Issues
1. **CRITICAL-2**: Pocket-ID security hardening (SSO provider)
2. **CRITICAL-1**: Jellyfin security hardening (media server)
3. **CRITICAL-0**: Network policies (cluster-wide)

### Branch Status
- Branch `security/ntfy-critical-hardening` can be deleted (merged to main)
