# Longhorn

This directory contains the manifests for deploying [Longhorn](https://longhorn.io/).

Longhorn is a lightweight, reliable, and easy-to-use distributed block storage system for Kubernetes.

## Helm Chart

This deployment uses the `longhorn` Helm chart from the [Longhorn chart repository](https://charts.longhorn.io/).

## StorageClass

This deployment defines a `StorageClass` named `longhorn-single` with the number of replicas set to 1. This is suitable for workloads where high availability is not a requirement.

## Troubleshooting

### Node disk shows DiskPressure / stops scheduling new replicas

**Symptom:** a PVC-backed app becomes hard to schedule, resize, or rebuild
replicas for. `kubectl get nodes.longhorn.io -n longhorn-system` still shows
the node `READY`, but the underlying disk is unschedulable.

**How this was diagnosed on 2026-07-21 (`talos-worker2`):**

1. Checked the Longhorn `Node` CR for the affected node and looked at
   `status.diskStatus.<disk>.conditions`:

   ```sh
   kubectl get nodes.longhorn.io talos-worker2 -n longhorn-system -o yaml \
     | grep -A8 "type: Schedulable"
   ```

   This surfaced a `DiskPressure` condition: `Schedulable: False`, with a
   message giving the exact numbers — `CurrentAvailable` vs `MinimalAvailable`
   (Longhorn's default is 25% of disk capacity must stay free before it will
   schedule new replicas there).

2. Compared `storageMaximum`/`storageAvailable` across all nodes sharing the
   same StorageClass to confirm it was isolated to one node, not a
   cluster-wide capacity problem:

   ```sh
   for n in talos-worker1 talos-worker2 talos-worker3; do
     echo "$n:"; kubectl get nodes.longhorn.io "$n" -n longhorn-system \
       -o jsonpath='{range .status.diskStatus.*}storageMaximum={.storageMaximum} storageAvailable={.storageAvailable}{"\n"}{end}'
   done
   ```

   All three nodes had near-identical `storageScheduled` (the sum of Longhorn's
   own tracked replica sizes), but `talos-worker2`'s actual used space was
   ~200GB higher than what Longhorn thought it had scheduled there — a strong
   signal of untracked data sitting on disk outside Longhorn's bookkeeping.

3. Checked for orphaned replica directories — data left on disk that Longhorn
   detected but isn't attached to any live `Replica` object (this happens when
   a node reboot, kubelet restart, or instance-manager crash interrupts a
   replica rebuild/migration before the old copy is cleaned up; Longhorn does
   **not** auto-delete these by default, it just flags them for review):

   ```sh
   kubectl get orphans.longhorn.io -n longhorn-system
   ```

   Found 14 orphans, all on `talos-worker2`, all created at the same
   timestamp (a single event, not an ongoing leak).

4. Before deleting anything, verified each one was actually safe to remove:

   ```sh
   kubectl get orphans.longhorn.io -n longhorn-system -o json | jq -r '
     .items[] | [.metadata.name, .spec.parameters.DataName,
                  (.status.conditions[] | select(.type=="DataCleanable").status),
                  (.status.conditions[] | select(.type=="Error").status)] | @tsv'
   ```

   Confirmed every orphan had `DataCleanable: True` / `Error: False`, and
   cross-referenced each orphan's `DataName` (the PVC volume it belonged to)
   against `kubectl get pv` and `kubectl get replicas.longhorn.io`. Two
   categories turned up: stale leftover copies of volumes that are still
   alive and healthy (a current, different-suffixed replica already exists
   elsewhere for the same volume), and copies belonging to volumes that had
   been deleted entirely. Neither category is live data.

5. Quantified the actual disk footprint before deleting, by exec'ing into the
   `longhorn-manager` pod running on the affected node (it hostPath-mounts
   `/var/lib/longhorn`):

   ```sh
   kubectl exec -n longhorn-system <longhorn-manager-pod-on-that-node> \
     -c longhorn-manager -- du -sh /var/lib/longhorn/replicas/<DataName>
   ```

   Totaled ~27.6GB across the 14 directories (one alone, a stale copy of the
   Jellyfin config volume, was 20G).

6. Deleted the confirmed-safe orphans — this is what actually triggers
   Longhorn to remove the on-disk directory (deleting the CR alone doesn't
   instantly finish; a finalizer runs the real cleanup, which can take a
   minute or two for large directories):

   ```sh
   kubectl delete orphans.longhorn.io <name> -n longhorn-system
   ```

7. Verified: `kubectl get orphans.longhorn.io -n longhorn-system` returned
   empty, `storageAvailable` on the node's disk status rose from ~62GB to
   ~92GB, and the `Schedulable` condition flipped back to `True`.

**Note:** since Longhorn doesn't auto-delete orphaned replica data by
default (a deliberate safety choice — see Longhorn's `orphan-auto-deletion`
setting), this can recur after any node reboot or instance-manager restart,
including during Talos/Kubernetes upgrades. Worth checking
`kubectl get orphans.longhorn.io -n longhorn-system` as a post-upgrade
health check, not just when a disk actually fills up.
