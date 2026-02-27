# Kubernetes Configuration Enhancement Branches

**Generated**: 2026-01-12
**Repository**: kubernetes-configs (funland cluster)
**Base Branch**: main
**Total Branches Created**: 26

---

## Overview

Following a comprehensive security, performance, and best practices audit of the Talos Linux homelab cluster, 26 enhancement branches have been created to address identified issues across all applications.

**Audit Summary:**
- 5 CRITICAL security issues
- 12 HIGH priority issues
- 15 MEDIUM priority improvements
- 8 LOW priority enhancements

---

## Branch Organization

### 🔴 CRITICAL Priority (Immediate Action Required)

These branches address severe security vulnerabilities that should be fixed within 1 week:

#### 1. `security/vaultwarden-critical-hardening`
**Severity**: CRITICAL
**App**: vaultwarden (Password Manager)
**Issues Fixed**:
- Add pod and container security contexts (currently missing)
- Implement runAsNonRoot, drop all capabilities
- Add health probes (liveness and readiness)
- Configure seccomp profile

**Files to Modify**: `funland/vaultwarden/vaultwarden/deployment.yaml`

**Impact**: Password manager currently runs as root with no security hardening - highest value target in cluster.

---

#### 2. `security/ntfy-critical-hardening`
**Severity**: CRITICAL
**App**: ntfy (Notification Service)
**Issues Fixed**:
- Add complete security context (currently absent)
- Implement runAsNonRoot, drop all capabilities
- Add readiness probe (only has liveness)
- Configure seccomp profile

**Files to Modify**: `funland/ntfy/ntfy/deployment.yaml`

**Impact**: Public-facing service with no security hardening.

---

#### 3. `security/pocket-id-critical-hardening`
**Severity**: CRITICAL
**App**: pocket-id (SSO Provider)
**Issues Fixed**:
- Add pod and container security contexts
- Pin image tag to specific version (currently v2, should be v2.1.0)
- Implement security hardening
- Add automountServiceAccountToken: false

**Files to Modify**: `funland/pocket-id/pocket-id/deployment.yaml`

**Impact**: Identity provider running without security context.

---

#### 4. `security/jellyfin-critical-hardening`
**Severity**: CRITICAL
**App**: jellyfin (Media Server)
**Issues Fixed**:
- Uncomment and enable capability drop
- Add runAsUser, runAsGroup, fsGroup
- Implement runAsNonRoot
- Add health probes

**Files to Modify**: `funland/media/jellyfin/deployment.yaml`

**Impact**: Media server runs as root with commented-out security features.

---

#### 5. `security/vikunja-container-security`
**Severity**: HIGH
**App**: vikunja (Task Management)
**Issues Fixed**:
- Add container-level security context (pod-level exists)
- Implement capability dropping
- Add health probes
- Increase resource requests (48Mi → 128Mi)

**Files to Modify**: `funland/vikunja/vikunja/deployment.yaml`

**Impact**: Incomplete security context configuration.

---

### 🟠 HIGH Priority (2-4 Weeks)

These branches address significant security and reliability issues:

#### 6. `security/network-policies-cluster-wide`
**Severity**: CRITICAL/HIGH
**Scope**: All namespaces
**Issues Fixed**:
- Implement deny-by-default NetworkPolicies for all namespaces
- Create application-specific allow rules
- Enable zero-trust networking

**Namespaces to Cover**:
- vaultwarden (CRITICAL)
- authentik (CRITICAL)
- pocket-id (CRITICAL)
- n8n (HIGH)
- ntfy (HIGH)
- All other application namespaces

**New Files**: Create `network-policy.yaml` in each app directory

**Impact**: Currently ZERO network policies exist - any compromised pod can access all services.

---

#### 7. `reliability/pod-disruption-budgets`
**Severity**: HIGH
**Scope**: CloudNativePG clusters, ArgoCD, critical services
**Issues Fixed**:
- Add PodDisruptionBudgets for all 3-replica CNPG clusters (minAvailable: 2)
- Add PDB for ArgoCD server
- Ensure availability during node maintenance

**New Files**:
- `funland/cnpg/cnpg-clusters/*/pdb.yaml` (5 files)
- `funland/argocd/argocd/pdb.yaml`

**Impact**: Cluster upgrades could drain all replicas simultaneously, causing outages.

---

#### 8. `performance/cnpg-resource-limits-and-pdbs`
**Severity**: HIGH
**Scope**: All CloudNativePG clusters
**Issues Fixed**:
- Add resource requests/limits to all 5 CNPG clusters
- Change from BestEffort to Guaranteed QoS class
- Prevent OOM and eviction issues

**Files to Modify**: All `funland/cnpg/cnpg-clusters/*-cnpg-cluster.yaml`

**Recommended Resources**:
```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"
```

**Impact**: Databases have no resource constraints, risk eviction and OOM.

---

#### 9. `security/helm-charts-security-contexts`
**Severity**: HIGH
**Scope**: All Helm applications without security overrides
**Issues Fixed**:
- Add pod security context overrides to Helm values
- Implement container security contexts
- Configure seccomp profiles

**Files to Modify**:
- `funland/cert-manager/cert-manager/application.yaml`
- `funland/external-secrets/external-secrets/application.yaml`
- `funland/gateway-api/gateway-api/application.yaml`
- Other infrastructure Helm charts

**Impact**: Helm charts deployed with default security (often insufficient).

---

#### 10. `security/authentik-pod-security`
**Severity**: HIGH
**App**: authentik (Identity Provider)
**Issues Fixed**:
- Add pod security context to Helm values
- Configure container security context
- Add resource limits

**Files to Modify**: `funland/authentik/authentik/application.yaml`

**Impact**: Authentication provider lacks pod-level security hardening.

---

#### 11. `security/immich-pod-security`
**Severity**: HIGH
**App**: immich (Photo Management)
**Issues Fixed**:
- Add pod security context to Helm values
- Configure container security context
- Add resource limits

**Files to Modify**: `funland/immich/immich/application.yaml`

**Impact**: Photo management service lacks security hardening.

---

#### 12. `performance/monitoring-stack-resources`
**Severity**: HIGH
**Scope**: Prometheus, Grafana, Loki, Promtail, Alertmanager
**Issues Fixed**:
- Add resource requests/limits to all monitoring components
- Prevent Prometheus eviction (stores critical metrics)
- Configure persistent volume for Prometheus

**Files to Modify**:
- `funland/monitoring/prometheus/application.yaml`
- `funland/monitoring/grafana/application.yaml`
- `funland/monitoring/loki/application.yaml`
- `funland/monitoring/promtail/application.yaml`

**Impact**: Monitoring stack has no resource limits, at risk of eviction.

---

#### 13. `performance/horizontal-pod-autoscalers`
**Severity**: HIGH
**Scope**: immich, n8n, jellyfin
**Issues Fixed**:
- Create HPA resources for scalable workloads
- Enable automatic scaling based on CPU/memory

**New Files**:
- `funland/immich/immich/hpa.yaml`
- `funland/n8n/n8n/hpa.yaml`
- `funland/media/jellyfin/hpa.yaml`

**Impact**: No workloads can scale automatically under load.

---

#### 14. `reliability/health-probes-improvements`
**Severity**: HIGH
**Scope**: vaultwarden, ntfy, vikunja
**Issues Fixed**:
- Add missing readiness probes
- Add missing liveness probes
- Tune probe parameters (failureThreshold, initialDelaySeconds)

**Files to Modify**:
- `funland/vaultwarden/vaultwarden/deployment.yaml`
- `funland/ntfy/ntfy/deployment.yaml`
- `funland/vikunja/vikunja/deployment.yaml`
- `funland/pocket-id/pocket-id/deployment.yaml`

**Impact**: Services lack proper health checking for automated recovery.

---

#### 15. `reliability/argocd-high-availability`
**Severity**: HIGH
**App**: argocd
**Issues Fixed**:
- Uncomment and enable HA configuration
- Increase server replica count to 3
- Deploy redis-ha for session storage
- Add resource limits

**Files to Modify**: `funland/argocd/argocd/application.yaml`

**Impact**: GitOps platform is single point of failure.

---

#### 16. `config/standard-kubernetes-labels`
**Severity**: HIGH
**Scope**: All raw deployments
**Issues Fixed**:
- Replace custom labels with standard `app.kubernetes.io/*` labels
- Add version, component, managed-by labels
- Improve monitoring and cost allocation

**Files to Modify**: All deployment.yaml files without standard labels

**Impact**: Non-standard labels make monitoring and management difficult.

---

#### 17. `reliability/increase-replica-counts`
**Severity**: HIGH
**Scope**: ntfy, vaultwarden, pocket-id (critical services)
**Issues Fixed**:
- Increase replicas from 1 to 2 for HA
- Configure RollingUpdate strategy
- Note: Requires RWX storage or stateless configuration

**Files to Modify**:
- `funland/ntfy/ntfy/deployment.yaml`
- `funland/vaultwarden/vaultwarden/deployment.yaml`
- `funland/pocket-id/pocket-id/deployment.yaml`

**Impact**: Single replicas create single points of failure.

---

### 🟡 MEDIUM Priority (1-2 Months)

These branches provide important improvements:

#### 18. `config/service-account-token-hardening`
**Severity**: MEDIUM
**Scope**: All deployments not needing K8s API access
**Issues Fixed**:
- Add `automountServiceAccountToken: false`
- Reduce attack surface

**Files to Modify**: All deployment.yaml files (except operators, ArgoCD, monitoring agents)

**Impact**: Unnecessary service account tokens increase attack surface.

---

#### 19. `config/topology-spread-constraints`
**Severity**: MEDIUM
**Scope**: Multi-replica deployments
**Issues Fixed**:
- Add topologySpreadConstraints for pod distribution
- Prevent multiple replicas on same node

**Files to Modify**: All deployments with replicas > 1

**Impact**: Multiple replicas could be on same node, reducing HA.

---

#### 20. `config/priority-classes`
**Severity**: MEDIUM
**Scope**: Cluster-wide
**Issues Fixed**:
- Create PriorityClass resources
- Assign priorities to critical workloads
- Ensure databases and auth survive eviction

**New Files**:
- `funland/kube-system/priority-classes.yaml`

**Priority Classes**:
- critical-infrastructure (1000000): CNPG, authentik
- high-priority-services (100000): vaultwarden, pocket-id
- default (0): other services

**Impact**: No prioritization for eviction scenarios.

---

#### 21. `config/graceful-shutdown-improvements`
**Severity**: MEDIUM
**Scope**: Databases, background workers
**Issues Fixed**:
- Tune terminationGracePeriodSeconds
- Add preStop hooks for connection draining

**Files to Modify**:
- CNPG clusters (60-120s)
- n8n (60s for workflow completion)
- Other long-running processes

**Impact**: Default 30s may be too short for graceful shutdown.

---

#### 22. `config/storage-size-adjustments`
**Severity**: MEDIUM
**Scope**: ntfy, adguard PVCs
**Issues Fixed**:
- Increase ntfy-user-db from 1Mi to 100Mi
- Increase ntfy-cache from 1Mi to 1Gi
- Increase adguard-workdir from 1Gi to 5Gi

**Files to Modify**:
- `funland/ntfy/ntfy/pvc.yaml`
- `funland/adguard/adguard/pvc.yaml`

**Impact**: Current sizes are too small and will fill up.

---

#### 23. `config/prometheus-monitoring-annotations`
**Severity**: MEDIUM
**Scope**: Raw deployments
**Issues Fixed**:
- Add Prometheus scrape annotations
- Enable metrics collection

**Files to Modify**: All deployment.yaml files

**Annotations**:
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

**Impact**: Missing annotations prevent automatic metrics discovery.

---

#### 24. `performance/resource-limits-all-helm`
**Severity**: HIGH/MEDIUM
**Scope**: All Helm applications
**Issues Fixed**:
- Add comprehensive resource specifications to all Helm charts
- Consolidate resource configuration

**Files to Modify**: All `application.yaml` files with Helm charts

**Impact**: Helm applications lack resource constraints.

---

### 📘 Documentation

#### 25. `docs/configuration-documentation`
**Severity**: MEDIUM
**Scope**: Complex configurations
**Issues Fixed**:
- Add explanatory comments to complex YAML
- Document non-obvious configuration choices
- Explain why certain security features are disabled

**Files to Modify**:
- Jellyfin (commented capability drop)
- CNPG clusters (postInitSQL scripts)
- AdGuard (NET_BIND_SERVICE requirement)

**Impact**: Complex configurations lack context for future maintainers.

---

### 🗑️ Cleanup Branch

#### 26. `performance/cnpg-resource-limits`
**Status**: Duplicate (can be deleted)
**Superseded by**: `performance/cnpg-resource-limits-and-pdbs`

---

## Implementation Roadmap

### Phase 1: Critical Security (Week 1)
**Priority**: IMMEDIATE
**Branches**:
1. security/vaultwarden-critical-hardening
2. security/ntfy-critical-hardening
3. security/pocket-id-critical-hardening
4. security/jellyfin-critical-hardening
5. security/vikunja-container-security

**Estimated Effort**: 1-2 days

---

### Phase 2: Network Segmentation (Week 2)
**Priority**: CRITICAL
**Branches**:
1. security/network-policies-cluster-wide

**Estimated Effort**: 2-3 days

---

### Phase 3: High Availability (Weeks 3-4)
**Priority**: HIGH
**Branches**:
1. reliability/pod-disruption-budgets
2. performance/cnpg-resource-limits-and-pdbs
3. reliability/argocd-high-availability
4. reliability/increase-replica-counts

**Estimated Effort**: 1 week

---

### Phase 4: Resource Management (Week 5)
**Priority**: HIGH
**Branches**:
1. performance/monitoring-stack-resources
2. performance/resource-limits-all-helm
3. security/helm-charts-security-contexts
4. security/authentik-pod-security
5. security/immich-pod-security

**Estimated Effort**: 2-3 days

---

### Phase 5: Reliability Improvements (Week 6)
**Priority**: HIGH/MEDIUM
**Branches**:
1. performance/horizontal-pod-autoscalers
2. reliability/health-probes-improvements
3. config/standard-kubernetes-labels

**Estimated Effort**: 2-3 days

---

### Phase 6: Configuration Hardening (Weeks 7-8)
**Priority**: MEDIUM
**Branches**:
1. config/service-account-token-hardening
2. config/priority-classes
3. config/topology-spread-constraints
4. config/graceful-shutdown-improvements
5. config/storage-size-adjustments
6. config/prometheus-monitoring-annotations

**Estimated Effort**: 1 week

---

### Phase 7: Documentation (Ongoing)
**Priority**: MEDIUM
**Branches**:
1. docs/configuration-documentation

**Estimated Effort**: 1-2 days

---

## Quick Reference: Branch by Application

### adguard
- ✅ Security posture EXCELLENT (best in cluster)
- Needs: network-policies-cluster-wide, storage-size-adjustments

### argocd
- Needs: reliability/argocd-high-availability, performance/resource-limits-all-helm, security/network-policies-cluster-wide, reliability/pod-disruption-budgets

### authentik
- Needs: security/authentik-pod-security, security/network-policies-cluster-wide, performance/resource-limits-all-helm, reliability/pod-disruption-budgets (CNPG)

### cert-manager
- Needs: security/helm-charts-security-contexts, performance/resource-limits-all-helm, security/network-policies-cluster-wide

### CloudNativePG (all clusters)
- Needs: performance/cnpg-resource-limits-and-pdbs, security/network-policies-cluster-wide, config/priority-classes

### external-secrets
- Needs: security/helm-charts-security-contexts, performance/resource-limits-all-helm

### gateway (Cilium)
- Needs: security/network-policies-cluster-wide, performance/resource-limits-all-helm

### immich
- Needs: security/immich-pod-security, performance/horizontal-pod-autoscalers, security/network-policies-cluster-wide

### jellyfin
- Needs: security/jellyfin-critical-hardening, performance/horizontal-pod-autoscalers, security/network-policies-cluster-wide

### longhorn
- Needs: performance/resource-limits-all-helm

### monitoring (Prometheus, Grafana, Loki, etc.)
- Needs: performance/monitoring-stack-resources, security/network-policies-cluster-wide

### n8n
- Needs: performance/horizontal-pod-autoscalers, security/network-policies-cluster-wide, reliability/increase-replica-counts, config/standard-kubernetes-labels

### ntfy
- Needs: security/ntfy-critical-hardening, reliability/health-probes-improvements, reliability/increase-replica-counts, config/storage-size-adjustments, security/network-policies-cluster-wide

### pocket-id
- Needs: security/pocket-id-critical-hardening, reliability/increase-replica-counts, security/network-policies-cluster-wide, config/standard-kubernetes-labels

### privatebin
- ✅ Security posture EXCELLENT
- Needs: security/network-policies-cluster-wide

### vaultwarden
- Needs: security/vaultwarden-critical-hardening, reliability/health-probes-improvements, reliability/increase-replica-counts, security/network-policies-cluster-wide, config/priority-classes

### vikunja
- Needs: security/vikunja-container-security, reliability/health-probes-improvements, security/network-policies-cluster-wide

---

## Testing Strategy

### Security Branches
1. Deploy to test namespace first
2. Verify pods start successfully with new security contexts
3. Test application functionality
4. Check for permission-denied errors in logs

### Network Policy Branches
1. Apply deny-all policy first
2. Test that connectivity is blocked
3. Apply allow rules incrementally
4. Verify service-to-service communication

### Resource Limit Branches
1. Monitor current resource usage before changes
2. Apply new limits with headroom
3. Watch for OOMKilled or throttling
4. Adjust based on actual usage

### HA Branches
1. Test rolling updates
2. Verify zero-downtime deployments
3. Test node drain scenarios
4. Validate PDB behavior during maintenance

---

## Notes

- **Duplicate Branch**: `performance/cnpg-resource-limits` can be deleted (superseded by `performance/cnpg-resource-limits-and-pdbs`)
- **Pre-existing Branch**: `security-hardening` already exists (not created by this process)
- All branches are based on `main` and ready for development
- No changes have been committed yet - branches are empty and ready for work
- Use ArgoCD diff feature to preview changes before merging
- Test each branch in isolated namespace where possible

---

## Commands for Branch Management

### View all enhancement branches:
```bash
git branch | grep -E "(security|reliability|performance|config|docs)/"
```

### Switch to a branch:
```bash
git checkout security/vaultwarden-critical-hardening
```

### Push all branches to remote:
```bash
git push -u origin --all
```

### Delete the duplicate branch:
```bash
git branch -D performance/cnpg-resource-limits
```

---

**Total Estimated Effort for Full Implementation**: 6-8 weeks

**Critical Path**: Phases 1-3 (security hardening and network policies) - 3-4 weeks

**Recommended Start Order**:
1. vaultwarden (password manager)
2. authentik/pocket-id (auth services)
3. ntfy (public-facing)
4. network-policies-cluster-wide
5. All other branches

---

Generated by kubernetes-optimizer agent analysis of 81 YAML manifests across 24 applications.
