# Hub Controller for Lighthouse

## Summary

The current Lighthouse implementation has a passive broker, it is a central apiserver that is
accessed by each cluster in the clusterset. Clusters synchronise resources to and from the
broker with all sync and reconciliation logic implemented in the clusters.

There are feature gaps that would be much easier to address if we introduce an active broker
that can implement reconciliation logic.

- `ServiceExport` conflict resolution, per the MCS KEP [^1]
- Merging exported `Services` into single `ServiceImport`

Benefits of central reconciliation in an active broker include:

- Single consistent model of intended state, rather than models computed per cluster
- Target state computed _before_ it gets synced out to clusters
- Synchronise fewer resources from broker to each cluster

## Proposal

This enhancement proposes to add an mcs-controller to the broker that will implement the
reconciliation logic from `ServiceExport` to `ServiceImport` resources. This will result in a
clear separation of logic for MCS resources:

- Clusters will sync resources to and from the broker
- The mcs-controller on the broker will reconcile `ServiceImport` object state from the
  provided `ServiceExport` state.

Once an mcs-controller has been introduced, the reconciliation logic will be extended to address
the following open issues:

- Mark conflicting `ServiceExport` objects with the `ServiceExportConflict` condition
- Merge matching `ServiceExport` objects into a single `ServiceImport` object

The resulting solution is expected to be less complex overall, ready to be more conformant to
the MCS KEP and to provide opportunity for minimising the data synchronisation between clusters.

[^1]: [KEP-1645: Multi-Cluster Services API](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api#constraints-and-conflict-resolution)
