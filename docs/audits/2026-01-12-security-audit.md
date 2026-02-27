# Kubernetes Configuration Security & Best Practices Audit Report

**Cluster**: funland (Talos Linux)
**Date**: 2026-01-12
**Audited by**: Kubernetes Platform Engineer
**Total Files Analyzed**: 81 YAML manifests
**Applications Reviewed**: 24 (8 raw deployments, 16 Helm charts)

---

## Executive Summary

### Overall Security Posture: MODERATE-HIGH RISK

The cluster demonstrates several strong practices (CloudNativePG with backups, External Secrets integration, some excellent security contexts) but has **critical security gaps** that require immediate attention:

- **5 CRITICAL issues** requiring immediate remediation
- **12 HIGH priority** issues requiring prompt attention
- **15 MEDIUM priority** improvements recommended
- **8 LOW priority** nice-to-have enhancements

### Key Strengths
- Excellent secret management via External Secrets + 1Password
- CloudNativePG clusters well-configured with encryption and backups
- Some deployments (AdGuard, PrivateBin) have exemplary security contexts
- Resource limits defined for most workloads
- Health probes present on most services

### Critical Gaps
- **ZERO network policies** across entire cluster (zero-trust networking absent)
- **NO pod disruption budgets** (availability risk during node maintenance)
- Multiple deployments missing security contexts or running as root
- Helm charts lack pod-level security configurations
- No HPA for any workload (no auto-scaling)

---

## Critical Issues (Immediate Action Required)

### CRITICAL-1: Complete Absence of Network Policies
**Severity**: CRITICAL
**Risk**: Pod-to-pod traffic is unrestricted. Compromised pod can access any service in cluster.
**Location**: All namespaces

**Current State**: Zero NetworkPolicy resources detected across entire cluster.

**Impact**:
- Attacker who compromises one pod can laterally move to databases, secrets, or other services
- No defense-in-depth against container breakout
- Violates zero-trust networking principles
- Compliance failure for most security frameworks (PCI-DSS, SOC2, etc.)

**Remediation**:

For each application namespace, implement deny-by-default policies:

```yaml
# Example for vaultwarden namespace
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: vaultwarden
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: vaultwarden-allow
  namespace: vaultwarden
spec:
  podSelector:
    matchLabels:
      app: vaultwarden
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: gateway
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:  # Allow DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  - to:  # External SMTP
    ports:
    - protocol: TCP
      port: 465
```

**Priority**: Implement for critical services first: vaultwarden, authentik, n8n, pocket-id

---

### CRITICAL-2: Jellyfin Running Without Proper Security Context
**Severity**: CRITICAL
**Risk**: Container runs as root with full capabilities
**Location**: `/home/myles/kubernetes-configs/funland/media/jellyfin/deployment.yaml`

**Issues**:
- No `runAsNonRoot: true`
- No `runAsUser` specified (defaults to root)
- Capabilities not dropped (ALL capabilities available)
- `readOnlyRootFilesystem` not set

**Current Configuration** (lines 26-51):
```yaml
containers:
- image: jellyfin/jellyfin:10.11.5
  name: jellyfin
  securityContext:
    allowPrivilegeEscalation: false
    # capabilities:
    #   drop: [ALL]  # COMMENTED OUT!
    seccompProfile:
      type: RuntimeDefault
```

**Remediation**:
```yaml
containers:
- image: jellyfin/jellyfin:10.11.5
  name: jellyfin
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    allowPrivilegeEscalation: false
    capabilities:
      drop: [ALL]
    readOnlyRootFilesystem: false  # Jellyfin needs write access to /config and /cache
    seccompProfile:
      type: RuntimeDefault
spec:
  securityContext:
    fsGroup: 1000
```

---

### CRITICAL-3: Ntfy Missing Security Context Entirely
**Severity**: CRITICAL
**Risk**: Container runs as root, full capabilities, no security hardening
**Location**: `/home/myles/kubernetes-configs/funland/ntfy/ntfy/deployment.yaml`

**Issues**:
- NO `securityContext` at pod or container level
- Runs as root (UID 0) by default
- All Linux capabilities available
- No seccomp profile

**Current Configuration** (lines 14-42):
```yaml
containers:
- name: ntfy
  image: binwiederhier/ntfy:v2.15
  args: ["serve"]
  # NO SECURITY CONTEXT AT ALL
```

**Remediation**:
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
  - name: ntfy
    image: binwiederhier/ntfy:v2.15
    args: ["serve"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
      readOnlyRootFilesystem: false  # ntfy needs write to /var/lib/ntfy
      seccompProfile:
        type: RuntimeDefault
```

---

### CRITICAL-4: Vaultwarden Missing Security Context
**Severity**: CRITICAL
**Risk**: Password manager running as root - highest value target
**Location**: `/home/myles/kubernetes-configs/funland/vaultwarden/vaultwarden/deployment.yaml`

**Issues**:
- NO `securityContext` defined (most critical app in cluster!)
- Stores encrypted password vaults but runs with root privileges
- No capability dropping
- No seccomp profile

**Remediation**:
```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      containers:
      - name: vaultwarden
        image: vaultwarden/server:1.35.2
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: [ALL]
          readOnlyRootFilesystem: false
          seccompProfile:
            type: RuntimeDefault
```

---

### CRITICAL-5: Pocket-ID (SSO Provider) Missing Security Context
**Severity**: CRITICAL
**Risk**: Identity provider running as root
**Location**: `/home/myles/kubernetes-configs/funland/pocket-id/pocket-id/deployment.yaml`

**Issues**:
- NO `securityContext` defined
- Handles authentication but runs with elevated privileges
- Image tag `v2` is imprecise (not semver-pinned)

**Remediation**:
```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      containers:
      - name: pocket-id
        image: ghcr.io/pocket-id/pocket-id:v2.1.0  # Pin exact version
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: [ALL]
          readOnlyRootFilesystem: false
          seccompProfile:
            type: RuntimeDefault
```

---

## High Priority Issues

### HIGH-1: Vikunja Missing Container-Level Security Context
**Severity**: HIGH
**Location**: `/home/myles/kubernetes-configs/funland/vikunja/vikunja/deployment.yaml`

**Issue**: Pod security context exists but no container-level hardening:
```yaml
spec:
  containers:
  - name: main
    image: vikunja/vikunja:0.24.6
    # NO container securityContext
```

**Remediation**:
```yaml
containers:
- name: main
  image: vikunja/vikunja:0.24.6
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop: [ALL]
    readOnlyRootFilesystem: false
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
```

---

### HIGH-2: Zero Pod Disruption Budgets
**Severity**: HIGH
**Risk**: No availability guarantees during node maintenance or cluster upgrades
**Location**: All namespaces

**Impact**:
- Cluster upgrades could drain all replicas simultaneously
- Node maintenance could cause service outages
- CloudNativePG clusters (3 replicas) could lose quorum

**Remediation Priority**:
1. **Critical services**: CloudNativePG clusters, ArgoCD, authentik
2. **High availability apps**: monitoring stack

```yaml
# Example for CloudNativePG
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: authentik-dbc-pdb
  namespace: postgres
spec:
  minAvailable: 2  # Maintain quorum for 3-replica cluster
  selector:
    matchLabels:
      cnpg.io/cluster: authentik-dbc
---
# Example for ArgoCD
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: argocd-server-pdb
  namespace: argocd
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server
```

---

### HIGH-3: No Horizontal Pod Autoscalers
**Severity**: HIGH
**Risk**: Manual scaling only, no response to load spikes
**Location**: All deployments

**Impact**:
- Services like immich, jellyfin, n8n cannot scale under load
- Resource waste (over-provisioning) or poor performance (under-provisioning)

**Recommended for**:
- **immich**: Photo processing can be CPU-intensive
- **n8n**: Workflow execution scales with job count
- **jellyfin**: Transcoding workload varies

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: immich-hpa
  namespace: immich
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: immich-server
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

### HIGH-4: Ntfy Missing Readiness Probe
**Severity**: HIGH
**Risk**: Pod marked ready before actually serving traffic
**Location**: `/home/myles/kubernetes-configs/funland/ntfy/ntfy/deployment.yaml`

**Issue**: Only liveness probe defined (line 25-32), no readiness probe.

**Impact**:
- Traffic sent to pod before ready
- Failed deployments may not be detected
- Potential connection errors during rolling updates

**Remediation**:
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

---

### HIGH-5: Vaultwarden Missing Health Probes
**Severity**: HIGH
**Risk**: No automated recovery if service hangs or crashes
**Location**: `/home/myles/kubernetes-configs/funland/vaultwarden/vaultwarden/deployment.yaml`

**Impact**:
- Kubernetes won't restart unhealthy pods
- Users experience downtime without automatic recovery

**Remediation**:
```yaml
livenessProbe:
  httpGet:
    path: /alive
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /alive
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

---

### HIGH-6: Missing Standard Kubernetes Labels
**Severity**: HIGH
**Risk**: Difficult monitoring, cost allocation, and resource management
**Location**: Multiple deployments (jellyfin, ntfy, vaultwarden, n8n, pocket-id)

**Issue**: Not using recommended `app.kubernetes.io/*` labels.

**Current** (jellyfin example):
```yaml
labels:
  app: jellyfin
```

**Recommended**:
```yaml
labels:
  app.kubernetes.io/name: jellyfin
  app.kubernetes.io/instance: jellyfin
  app.kubernetes.io/version: "10.11.5"
  app.kubernetes.io/component: media-server
  app.kubernetes.io/part-of: homelab
  app.kubernetes.io/managed-by: argocd
```

---

### HIGH-7: No Resource Requests for Prometheus
**Severity**: HIGH
**Risk**: Prometheus can be evicted under memory pressure, losing metrics
**Location**: `/home/myles/kubernetes-configs/funland/monitoring/prometheus/application.yaml`

**Issue**: Helm chart deployed with default values (no resource configuration).

**Impact**:
- BestEffort QoS class (first to be evicted)
- Unpredictable resource usage
- Potential OOM kills during high cardinality scenarios

**Remediation**:
```yaml
helm:
  valuesObject:
    server:
      resources:
        requests:
          cpu: 500m
          memory: 2Gi
        limits:
          cpu: 2000m
          memory: 4Gi
      persistentVolume:
        size: 50Gi
    alertmanager:
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 200m
          memory: 256Mi
```

---

### HIGH-8: Grafana/Loki/Promtail Lack Resource Constraints
**Severity**: HIGH
**Location**:
- `/home/myles/kubernetes-configs/funland/monitoring/grafana/application.yaml`
- `/home/myles/kubernetes-configs/funland/monitoring/loki/application.yaml`
- `/home/myles/kubernetes-configs/funland/monitoring/promtail/application.yaml`

**Issue**: All monitoring Helm charts deployed without resource specifications.

**Remediation**: Add resource blocks to each application's valuesObject section.

---

### HIGH-9: Authentik and Immich Helm Charts Missing Security Contexts
**Severity**: HIGH
**Location**:
- `/home/myles/kubernetes-configs/funland/authentik/authentik/application.yaml`
- `/home/myles/kubernetes-configs/funland/immich/immich/application.yaml`

**Issue**: Helm charts deployed without pod security context overrides.

**Remediation** (authentik example):
```yaml
helm:
  valuesObject:
    server:
      podSecurityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containerSecurityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: [ALL]
        readOnlyRootFilesystem: false
```

---

### HIGH-10: CloudNativePG Clusters Missing Resource Limits
**Severity**: HIGH
**Location**: All `funland/cnpg/cnpg-clusters/*-cnpg-cluster.yaml` files

**Issue**: No resource requests/limits defined for PostgreSQL pods.

**Impact**:
- BestEffort QoS (evictable)
- Database could OOM under load
- No predictable performance guarantees

**Remediation** (add to each cluster spec):
```yaml
spec:
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "2000m"
```

---

### HIGH-11: Single Replica for Critical Services
**Severity**: HIGH
**Risk**: Single point of failure
**Location**:
- ntfy (public-facing notification service)
- vaultwarden (password manager)
- pocket-id (SSO provider)

**Issue**: `replicas: 1` for services that should be highly available.

**Impact**:
- Downtime during pod restarts
- No rolling updates possible
- Availability issues during node maintenance

**Remediation**:
```yaml
spec:
  replicas: 2  # Minimum for HA
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

Note: This requires either:
- Longhorn RWX storage class for PVCs, OR
- Removing stateful storage and using external backends

---

### HIGH-12: ArgoCD Single Replica Without HA
**Severity**: HIGH
**Location**: `/home/myles/kubernetes-configs/funland/argocd/argocd/application.yaml`

**Issue**: Lines 75-86 show HA configuration commented out.

**Recommendation**: Enable HA mode:
```yaml
server:
  replicaCount: 3
  database:
    host: argocd-redis-ha.argocd-redis-ha.svc
    port: 6379
# Deploy redis-ha separately using Helm chart
redis:
  enabled: false  # Disable built-in Redis
```

---

## Medium Priority Improvements

### MEDIUM-1: Missing automountServiceAccountToken: false
**Severity**: MEDIUM
**Location**: All deployments

**Issue**: Service account tokens mounted by default, increasing attack surface.

**Remediation**: Add to all deployment specs that don't need Kubernetes API access:
```yaml
spec:
  template:
    spec:
      automountServiceAccountToken: false
```

**Exceptions** (needs API access): ArgoCD, monitoring agents, operators.

---

### MEDIUM-2: Imprecise Image Tag (pocket-id:v2)
**Severity**: MEDIUM
**Location**: `/home/myles/kubernetes-configs/funland/pocket-id/pocket-id/deployment.yaml`

**Issue**: `image: ghcr.io/pocket-id/pocket-id:v2` - not semver-pinned.

**Remediation**: Use full semver tag like `v2.1.0` or digest pinning.

---

### MEDIUM-3: Missing Topology Spread Constraints
**Severity**: MEDIUM
**Location**: All multi-replica deployments

**Issue**: No pod distribution across failure domains.

**Impact**: Multiple replicas could be scheduled on same node.

**Remediation** (for future HA deployments):
```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: vaultwarden
```

---

### MEDIUM-4: No Pod Priority Classes
**Severity**: MEDIUM
**Location**: All deployments

**Issue**: No priority classes defined to ensure critical workloads survive eviction.

**Remediation**: Create priority classes and assign:
```yaml
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-infrastructure
value: 1000000
globalDefault: false
description: "Critical infrastructure components (databases, auth)"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-services
value: 100000
globalDefault: false
description: "User-facing services"
```

Then add to critical pods:
```yaml
spec:
  priorityClassName: critical-infrastructure
```

---

### MEDIUM-5: Missing terminationGracePeriodSeconds Tuning
**Severity**: MEDIUM
**Location**: All deployments

**Issue**: Default 30s may be too short for graceful shutdown of databases.

**Recommendation**:
- **Databases (CNPG)**: 60-120 seconds
- **Web services**: 30 seconds (default is fine)
- **Background workers (n8n)**: 60 seconds

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60
```

---

### MEDIUM-6: No preStop Hooks
**Severity**: MEDIUM
**Location**: All deployments

**Issue**: No graceful connection draining before shutdown.

**Remediation** (web services example):
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]  # Allow time for load balancer to update
```

---

### MEDIUM-7: Insufficient Liveness Probe Parameters
**Severity**: MEDIUM
**Location**: Multiple deployments

**Issue**: Some liveness probes have aggressive settings that could cause flapping.

Example: pocket-id has `failureThreshold: 2` with 90s period (line 52).

**Recommendation**: Use at least `failureThreshold: 3` and tune `initialDelaySeconds` based on startup time.

---

### MEDIUM-8: Missing Startup Probes for Slow-Starting Apps
**Severity**: MEDIUM
**Location**: immich, authentik (via Helm)

**Issue**: Apps with slow initialization should use startup probes.

**Remediation**:
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10  # Allow up to 5 minutes for startup
```

---

### MEDIUM-9: No Update Strategy Configuration
**Severity**: MEDIUM
**Location**: Multiple deployments

**Issue**: Missing `maxUnavailable` and `maxSurge` for rolling updates.

**Recommendation**:
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0  # For critical services
      maxSurge: 1
```

---

### MEDIUM-10: PVC Storage Sizes May Be Undersized
**Severity**: MEDIUM
**Location**:
- ntfy PVCs: 1Mi (extremely small, likely to fill up)
- AdGuard PVCs: 1Gi (may be too small for logs)

**Recommendation**:
- **ntfy-user-db**: 100Mi (currently 1Mi)
- **ntfy-cache**: 1Gi (currently 1Mi)
- **adguard-workdir**: 5Gi (logs can grow large)

---

### MEDIUM-11: Missing Resource Request Tuning
**Severity**: MEDIUM
**Location**: vikunja, privatebin

**Issue**: Resource requests may be too low, causing performance issues.

**vikunja**: 48Mi memory request may cause OOM during usage spikes.
**Recommendation**: Increase to 128Mi request, 256Mi limit.

---

### MEDIUM-12: No Monitoring Annotations
**Severity**: MEDIUM
**Location**: Raw deployment manifests

**Issue**: Missing Prometheus scrape annotations:
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

---

### MEDIUM-13: Missing Affinity Rules for Database Pods
**Severity**: MEDIUM
**Location**: CloudNativePG clusters

**Current**: Only anti-affinity configured (line 13-16 in clusters).

**Recommendation**: Add node affinity to prefer storage-optimized nodes if available:
```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: node-role.kubernetes.io/storage
          operator: In
          values:
          - "true"
```

---

### MEDIUM-14: Jellyfin CPU Limits Too Restrictive
**Severity**: MEDIUM
**Location**: `/home/myles/kubernetes-configs/funland/media/jellyfin/deployment.yaml`

**Issue**: 1000m (1 core) CPU limit for transcoding workload.

**Impact**: Transcoding will be throttled, poor streaming performance.

**Recommendation**:
```yaml
resources:
  requests:
    cpu: "500m"
    memory: 3.5Gi
  limits:
    cpu: "4000m"  # Allow burst for transcoding
    memory: 4Gi
```

---

### MEDIUM-15: No Documentation Comments in Complex Configurations
**Severity**: MEDIUM
**Location**: Multiple files

**Issue**: Complex configurations lack explanatory comments.

**Examples**:
- AdGuard requires NET_BIND_SERVICE capability (line 58) - well documented ✓
- Jellyfin has commented-out capability drop (line 48-49) - unclear why
- CloudNativePG postInitSQL (lines 42-46) - well documented ✓

**Recommendation**: Add comments explaining non-obvious choices.

---

## Low Priority / Nice-to-Have

### LOW-1: Consider ReadOnlyRootFilesystem Where Possible
**Severity**: LOW
**Location**: n8n, pocket-id, vikunja

**Current**: `readOnlyRootFilesystem: false` or not set.

**Recommendation**: Test if applications work with read-only root and emptyDir volumes for temp files.

---

### LOW-2: Consider Image Digest Pinning
**Severity**: LOW
**Location**: All deployments

**Current**: Tag-based image references (e.g., `v0.107.71`).

**Enhancement**: Pin by digest for immutability:
```yaml
image: adguard/adguardhome:v0.107.71@sha256:abc123...
```

---

### LOW-3: Add Node Selectors for Workload Separation
**Severity**: LOW

**Enhancement**: If cluster grows, separate workloads by node pools:
```yaml
nodeSelector:
  workload-type: media  # For jellyfin
  workload-type: database  # For CNPG
  workload-type: compute  # For n8n
```

---

### LOW-4: Consider Cluster-Wide SecurityContext Defaults
**Severity**: LOW

**Enhancement**: Use Pod Security Standards or Kyverno to enforce defaults:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: enforce
  rules:
  - name: check-runAsNonRoot
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Pods must run as non-root"
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true
```

---

### LOW-5: Consolidate Namespace Definitions
**Severity**: LOW
**Location**: Multiple files

**Issue**: Namespaces defined inline with deployments.

**Enhancement**: Create dedicated `namespace.yaml` files for consistency (some already exist like vikunja, n8n).

---

### LOW-6: Add Renovate Annotations for Custom Patterns
**Severity**: LOW

**Enhancement**: Ensure Renovate can auto-update all image tags:
```yaml
# renovate: datasource=docker depName=ghcr.io/pocket-id/pocket-id
image: ghcr.io/pocket-id/pocket-id:v2.1.0
```

---

### LOW-7: Consider Longhorn RWX StorageClass for HA
**Severity**: LOW
**Location**: PVCs using RWO

**Current**: Services like ntfy, vaultwarden use ReadWriteOnce, preventing multi-replica.

**Enhancement**: Migrate to RWX-capable storage class to enable HA.

---

### LOW-8: Add Grafana Dashboards via ConfigMaps
**Severity**: LOW

**Enhancement**: Pre-provision dashboards for CloudNativePG, Cilium, Longhorn via Grafana Helm values.

---

## Per-Namespace Detailed Findings

### adguard
**Files**: deployment.yaml, service.yaml, http-route.yaml
**Overall**: EXCELLENT security posture

**Findings**:
- ✅ Comprehensive security context (best in cluster)
- ✅ Resource limits defined
- ✅ Health probes configured
- ✅ Capabilities properly scoped (NET_BIND_SERVICE only)
- ⚠️ Missing: NetworkPolicy, PodDisruptionBudget
- ⚠️ Single replica (acceptable for DNS caching use case)

---

### argocd
**Files**: application.yaml, appset.yaml, http-route.yaml
**Overall**: GOOD, HA needs configuration

**Findings**:
- ⚠️ HA configuration commented out (lines 75-86)
- ⚠️ No resource limits specified in Helm values
- ❌ No NetworkPolicy
- ⚠️ Single Redis instance (should be redis-ha for HA setup)
- ✅ SSO integration via authentik
- ✅ RBAC configured

---

### authentik
**Files**: application.yaml, http-route.yaml, cnpg-cluster.yaml
**Overall**: MODERATE, missing pod-level security

**Findings**:
- ❌ Helm chart deployed without pod security context overrides
- ❌ No resource limits in Helm values
- ✅ External PostgreSQL (CNPG) with backup
- ✅ Redis enabled for sessions
- ❌ No NetworkPolicy
- ⚠️ No PodDisruptionBudget

---

### certificate / cert-manager
**Files**: application.yaml, cluster-issuer.yaml, wildcard.yaml
**Overall**: GOOD

**Findings**:
- ✅ Wildcard cert for *.batlab.io
- ✅ Cloudflare DNS-01 challenge
- ✅ Let's Encrypt integration
- ⚠️ No resource limits in cert-manager Helm chart
- ❌ No NetworkPolicy

---

### cnpg
**Files**: Multiple cluster definitions, scheduled-backups.yaml
**Overall**: EXCELLENT (best practice for HA databases)

**Findings**:
- ✅ 3 replicas for quorum
- ✅ Pod anti-affinity (required)
- ✅ Backups to S3 (Backblaze B2)
- ✅ WAL and data encryption (AES256)
- ✅ 30-day retention
- ✅ Monitoring enabled (PodMonitor)
- ✅ Security-hardened bootstrap SQL
- ❌ Missing: Resource limits for PostgreSQL pods
- ❌ Missing: PodDisruptionBudget (minAvailable: 2)
- ❌ Missing: NetworkPolicy

---

### external-secrets
**Files**: application.yaml, cluster-secret-store.yaml
**Overall**: EXCELLENT

**Findings**:
- ✅ 1Password integration working
- ✅ ClusterSecretStore for all namespaces
- ✅ PushSecret pattern for database credentials
- ⚠️ No resource limits in Helm values

---

### gateway
**Files**: internal-gateway.yaml, external-gateway.yaml, http-route.yaml
**Overall**: GOOD

**Findings**:
- ✅ Separate internal/external gateways
- ✅ L2 IP announcements (192.168.10.30-31)
- ✅ TLS termination with wildcard cert
- ✅ Gateway API (modern, not legacy Ingress)
- ⚠️ No NetworkPolicy to restrict gateway access

---

### immich
**Files**: application.yaml, http-route.yaml, cnpg-cluster.yaml
**Overall**: MODERATE

**Findings**:
- ⚠️ Helm chart without pod security context
- ⚠️ No resource limits specified
- ✅ NFS storage for media library
- ✅ External PostgreSQL (CNPG)
- ✅ Valkey (Redis) enabled
- ❌ No NetworkPolicy
- ⚠️ No HPA (photo processing can be CPU-intensive)

---

### jellyfin (media namespace)
**Files**: deployment.yaml, service.yaml, http-route.yaml
**Overall**: POOR security

**Findings**:
- ❌ Running as root (no runAsUser)
- ❌ Capabilities drop commented out
- ✅ Resource limits defined
- ✅ NFS mount for media (read-only)
- ❌ No health probes
- ❌ No NetworkPolicy
- ⚠️ CPU limit too low for transcoding (1 core)

---

### kube-system (cilium)
**Files**: application.yaml, l2-config.yaml, http-route.yaml
**Overall**: GOOD

**Findings**:
- ✅ Cilium CNI with Gateway API
- ✅ L2 announcements for LoadBalancer IPs
- ✅ IP pool configured (192.168.10.30-49)
- ⚠️ No resource limits in Cilium Helm values
- Note: Cilium requires elevated privileges (expected)

---

### longhorn-system
**Files**: application.yaml, http-route.yaml, longhorn-single-sc.yaml
**Overall**: GOOD

**Findings**:
- ✅ Distributed block storage
- ✅ Custom storage class (longhorn-single) for CNPG
- ✅ Web UI exposed via HTTPRoute
- ⚠️ No resource limits in Helm values
- Note: Longhorn requires privileged mode (expected)

---

### monitoring
**Files**: prometheus, grafana, loki, promtail, metrics-server applications
**Overall**: MODERATE

**Findings**:
- ❌ NO resource limits on any monitoring component
- ❌ Prometheus at risk of eviction (stores metrics)
- ⚠️ No persistent volume configuration for Prometheus
- ⚠️ Grafana without security context overrides
- ❌ No NetworkPolicy
- ⚠️ No PodDisruptionBudget for Prometheus

---

### n8n
**Files**: deployment.yaml, service.yaml, configmap.yaml, external-secret.yaml
**Overall**: MODERATE

**Findings**:
- ✅ Pod security context (runAsUser, fsGroup)
- ⚠️ Missing container-level security context
- ✅ Resource limits defined
- ✅ Health probes configured
- ✅ External PostgreSQL (CNPG)
- ✅ ExternalSecret for credentials
- ❌ No NetworkPolicy
- ⚠️ Single replica (should be 2+ for HA)
- ⚠️ No HPA (workflow execution scales with jobs)

---

### ntfy
**Files**: deployment.yaml, http-route.yaml, external-secret.yaml
**Overall**: POOR

**Findings**:
- ❌ NO security context (critical issue)
- ✅ Resource limits defined
- ⚠️ Only liveness probe (missing readiness)
- ✅ ExternalSecret for config
- ❌ No NetworkPolicy
- ⚠️ Single replica (public-facing service)
- ⚠️ PVC only 1Mi (extremely small)

---

### pocket-id
**Files**: deployment.yaml, service.yaml, http-route.yaml, external-secret.yaml
**Overall**: POOR

**Findings**:
- ❌ NO security context (SSO provider!)
- ✅ Resource limits defined
- ✅ Health probes configured
- ✅ ExternalSecret for encryption key
- ❌ No NetworkPolicy
- ⚠️ Image tag imprecise (v2 instead of v2.1.0)
- ⚠️ Single replica (identity provider should be HA)

---

### privatebin
**Files**: deployment.yaml, service.yaml, http-route.yaml
**Overall**: EXCELLENT security

**Findings**:
- ✅ Comprehensive security context (second-best in cluster)
- ✅ Read-only root filesystem
- ✅ All capabilities dropped
- ✅ Resource limits with ephemeral storage
- ✅ Health probes configured
- ⚠️ RWX PVC (2Gi) - could use RWO if single replica
- ❌ No NetworkPolicy
- ⚠️ No ExternalSecret (no secrets needed - good!)

---

### vaultwarden
**Files**: deployment.yaml, service.yaml, http-route.yaml, external-secret.yaml
**Overall**: CRITICAL security issues

**Findings**:
- ❌ NO security context (password manager running as root!)
- ✅ Resource limits defined
- ❌ No health probes
- ✅ ExternalSecret for SMTP, admin token, DB credentials
- ✅ External PostgreSQL (CNPG)
- ❌ No NetworkPolicy (critical for password vault)
- ⚠️ Single replica (should be 2+ for HA)

---

### vikunja
**Files**: deployment.yaml, service.yaml, http-route.yaml, external-secret.yaml
**Overall**: MODERATE

**Findings**:
- ✅ Pod security context configured
- ⚠️ Missing container-level security context
- ⚠️ Resource requests too low (48Mi memory)
- ✅ External PostgreSQL (CNPG)
- ✅ SSO via config secret
- ❌ No health probes
- ❌ No NetworkPolicy

---

## Summary Statistics

### Files Analyzed
- **Total YAML files**: 81
- **Raw Deployments**: 8
- **Helm Applications**: 16
- **CloudNativePG Clusters**: 5
- **HTTPRoutes**: 13
- **ExternalSecrets**: 8

### Applications Reviewed by Category
- **Critical Infrastructure**: argocd, cert-manager, external-secrets, cilium, longhorn (5)
- **Authentication/Identity**: authentik, pocket-id, vaultwarden (3)
- **Databases**: CloudNativePG clusters (5 instances)
- **Monitoring**: prometheus, grafana, loki, promtail, metrics-server (5)
- **User Applications**: immich, jellyfin, n8n, ntfy, vikunja, privatebin (6)
- **Network Services**: adguard (1)

### Security Posture by Application
| Application | Security Grade | Key Issues |
|------------|---------------|------------|
| AdGuard | A | None - excellent |
| PrivateBin | A | None - excellent |
| CloudNativePG | A- | Missing resource limits, PDB |
| ArgoCD | B | Missing HA config, resources |
| n8n | B- | Incomplete security context |
| Authentik | C | No pod security overrides |
| Immich | C | No pod security overrides |
| Vikunja | C | Incomplete security, no probes |
| Monitoring Stack | C | No resource limits |
| Jellyfin | D | Running as root |
| Ntfy | F | No security context |
| Vaultwarden | F | No security context, no probes |
| Pocket-ID | F | No security context |

### Issues by Severity
- **CRITICAL**: 5 issues (immediate action required)
- **HIGH**: 12 issues (address within 1-2 weeks)
- **MEDIUM**: 15 improvements (address within 1-2 months)
- **LOW**: 8 enhancements (nice-to-have)

**Total Findings**: 40

### Estimated Remediation Effort

#### Phase 1: Critical Security Fixes (1-2 days)
- Add security contexts to: ntfy, vaultwarden, pocket-id, jellyfin (4 deployments)
- Test deployments after hardening

#### Phase 2: Network Policies (2-3 days)
- Design and implement deny-by-default policies for all namespaces
- Create application-specific allow rules
- Test connectivity after policy deployment

#### Phase 3: High Availability (1 week)
- Implement PodDisruptionBudgets for critical services
- Configure ArgoCD HA mode
- Increase replicas for public-facing services

#### Phase 4: Resource Management (2-3 days)
- Add resource limits to all Helm applications
- Add resource limits to CloudNativePG clusters
- Configure resource quotas per namespace

#### Phase 5: Observability & Reliability (1 week)
- Add missing health probes
- Implement HPAs for scalable workloads
- Add monitoring annotations
- Configure alerting in Prometheus

**Total Effort**: 2-3 weeks for full remediation

---

## Recommendations by Priority

### Immediate Actions (This Week)
1. Add security contexts to vaultwarden, pocket-id, ntfy (CRITICAL services)
2. Fix jellyfin security context
3. Implement NetworkPolicies for vaultwarden, authentik, pocket-id (auth/secrets)
4. Add health probes to vaultwarden

### Short-Term (2-4 Weeks)
1. Implement NetworkPolicies for all namespaces
2. Add PodDisruptionBudgets for CNPG clusters and ArgoCD
3. Add resource limits to all Helm applications
4. Configure ArgoCD HA mode
5. Increase replicas for critical services

### Medium-Term (1-2 Months)
1. Implement HPAs for scalable workloads
2. Add topology spread constraints
3. Implement priority classes
4. Tune probe parameters and grace periods
5. Add monitoring annotations

### Long-Term (Ongoing)
1. Consider Kyverno for policy enforcement
2. Implement image digest pinning
3. Set up Grafana dashboards
4. Review and optimize resource allocations quarterly

---

## Positive Findings Worth Highlighting

1. **Excellent Secret Management**: Full adoption of External Secrets with 1Password integration - no hardcoded secrets found.

2. **CloudNativePG Best Practices**: Database clusters are exceptionally well-configured with encryption, backups, monitoring, and anti-affinity.

3. **Gateway API Adoption**: Modern ingress using Gateway API with separate internal/external gateways.

4. **GitOps Excellence**: Clean ArgoCD ApplicationSet pattern with automated sync/prune.

5. **Some Exemplary Security**: AdGuard and PrivateBin demonstrate excellent security context configuration.

6. **Resource Limits Present**: Most raw deployments have resource limits defined.

7. **Health Probes Common**: Most services include at least liveness probes.

---

## Conclusion

Your Kubernetes cluster shows strong architectural decisions (GitOps, External Secrets, CloudNativePG, Gateway API) but has **critical security gaps** requiring immediate attention. The absence of NetworkPolicies and security contexts on several critical services (password manager, SSO provider, notification service) presents significant risk.

**Priority**: Address the 5 CRITICAL issues within 1 week, focusing on services handling authentication and secrets first.

The good news: Most issues are straightforward to fix with configuration changes. No major architectural redesign needed.

---

**Report Generated**: 2026-01-12
**Next Audit Recommended**: After critical/high priority issues resolved (4 weeks)
