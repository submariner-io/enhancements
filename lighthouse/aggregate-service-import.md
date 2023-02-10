# Aggregate Service Import for Lighthouse

## Status

Deferred

## Summary

The current Lighthouse implementation deviates from the MCS specification in the following ways:

- ServiceExports with matching namespace/name are not combined into a single ServiceImport.
  Each importing cluster receives multiple ServiceImports that are multiplexed at DNS lookup.
- Conflict resolution of ServiceExports is not implemented [^1].

In order to better align with the MCS specification the following functionality needs to be
added to the Lighthouse agent:

- Merge matching `ServiceExport` objects into a single `ServiceImport` object
- Mark conflicting `ServiceExport` objects with the `ServiceExportConflict` condition

There are changes required in Lighthouse's handling of `EndpointSlices` before aggregated
`ServiceImports` can be introduced.

Currently, each cluster exporting a service creates a `ServiceImport` containing its local service VIP and service ports.
However the individual cluster information cannot be aggregated into a single `ServiceImport`. The MCS specification
intends for cluster-specific information to be stored in `EndpointSlices`, with a separate `EndpointSlice` per exporting
cluster. Currently, Lighthouse does export `EndpointSlices` per cluster containing the local endpoints that are simply
used by the CoreDNS plugin to determine if the cluster's service is healthy. The `EndpointSlice` purpose will need to
change to contain the local service VIP and ports.

Headless services already have `EndpointSlices` containing the correct pod IP addresses, and
contain no VIP information in the multiple `ServiceImports`. A merged `ServiceImport` for
headless services just needs to list the originating clusters in its status.

## Proposal

Submariner has a passive broker, it is a central apiserver that is accessed by each cluster in
the clusterset. Clusters synchronise resources to and from the broker with all sync and
reconciliation logic implemented in the clusters.

In this proposal we retain the passive broker and take an approach to merging `ServiceImports`
where exporting clusters handle the merging. Simplistically, a merged `ServiceImport` outcome
can be achieved by allowing the first exporting cluster to create a `ServiceImport` and all
subsequent exporting clusters to reuse it, handling conflicts appropriately.

1. First exporting cluster creates a `ServiceImport` on the broker.
2. Subsequent exporting clusters reuse the existing `ServiceImport`, merging their information appropriately and updating
   the status to include the ID of the exporting cluster.
3. Exporting clusters create and sync `EndpointSlices` to the broker with the service VIP and a
   `multicluster.kubernetes.io/service-name` label that matches the `ServiceImport`.
4. All clusters sync the `ServiceImport` and `EndpointSlices` from the broker.

### Handling Conflicts

#### Service Ports

The MCS specification states that the derived service will expose the union of the service ports declared on its constituent
services. However this would seem problematic if a connection request for a service port is directed to a cluster that
does not serve that port. To prevent this scenario, Lighthouse will expose the intersection of all the constituent
service ports in the aggregated `ServiceImport`. If there are any non-matching service ports, each constituent cluster's
`ServiceExport` will have a `ServiceExportConflict` condition set in its status.

#### Service Type

The MCS specification states that a service type conflict will be resolved by assigning the service type of the oldest
`ServiceExport` to the derived service. However it doesn't specify the behavior of a constituent service in conflict with
the chosen service type, eg a `Headless` service cannot simply be exported as a `ClusterIP` service. In this case,
Lighthouse will set the `ServiceExportConflict` condition but not export the constituent service. The `Valid` condition
will still be set to true because the service itself, by definition, is still exportable.

### DNS Resolution

Lighthouse currently implements DNS resolution by round-robin load balancing between the
`ServiceImports` that match the namespace/name pair. With a merged `ServiceImport`, the
per-cluster service VIPs and port will be stored on the `EndpointSlices`. DNS resolution will select from
the `EndpointSlices` associated with the aggregated `ServiceImport`. If no specific cluster is requested, the
service ports from the aggregated `ServiceImport` will be returned, otherwise the requested cluster's service ports
from its `EndpointSlice` will be returned.

DNS Resolution behaviour for headless services will remain unchanged.

### Service VIPs in Importing Clusters

Given the way that Submariner operates, there is no need to assign a service VIP to a
`ServiceImport` in the importing clusters. This means that `ServiceImports` can always have an
empty IP address attribute, like they do for headless services today.

In future it would be possible to allocate a Service VIP to each `ServiceImport` and implement
load balancing across the exporting clusters. This would allow more sophisticated load balancing
than is possible with the DNS resolver.

### Adding a ServiceExport

Here are the detailed steps for adding a `ServiceExport` to a cluster:

1. When a new `ServiceExport` is created, a local `ServiceImport` is created in the submariner namespace containing the
   local `Service` information.
2. The local `ServiceImport` is checked for conflicts with the aggregated `ServiceImport` located on the broker cluster
   in the broker namespace.
3. If in conflict, a `ServiceExportConflict` condition is set in the `ServiceExport` status and conflict resolution is
   attempted.
4. If a conflict cannot be resolved, the cluster's service information is not exported.
5. Otherwise or if no conflicts, Lighthouse updates `ServiceImport.Status.Clusters` to include the cluster
   name and merges the port information appropriately.
6. The aggregated `ServiceImport` is synced from the broker to the namespace of the local service.
7. A new `EndpointSlice` is created containing the service VIP (or Pod VIPs for headless) and synced to
   the broker.

### Removing a ServiceExport

Here are the steps for removing a `ServiceExport` from a cluster:

1. When a `ServiceExport` is removed, the associated local `ServiceImport` and `EndpointSlice` are deleted.
2. The aggregated `ServiceImport` is updated to remove the cluster name and re-calculate the service ports based on the
   remaining clusters.
3. If the aggregated `ServiceImport` status has an empty cluster list, it is removed.
4. Other clusters with `ServiceExports` in conflict should watch for `ServiceImport` updates and re-evaluate
   any conflicts, possibly creating a new `ServiceImport` if the prior conflict prevented exporting.

### EndpointSlice Behaviour

The endpoint list for `EndpointSlice` that gets created by an exporting cluster will contain one of:

- The service VIP (or Globalnet service VIP) for ClusterIP services
- The list of pod IPs (or Globalnet pod IPs) for Headless services

When an exporting cluster has no pods backing an exported service, the endpoint list of the `EndpointSlice` for the
cluster will be cleared. This is to ensure that requests do not get directed to clusters that
have no pods to service the request.

### Migration

Per-cluster `ServiceImports` wlll no longer be created but, for rolling cluster upgrades, there may be a period
of time where there will be a mix of upgraded and non-upgraded clusters. The latter still need to observe the per-cluster
`ServiceImports` so they need to remain on the broker during the transition period.

An upgraded cluster can remove its local copy of a remote cluster's `ServiceImport` when it observes the remote cluster's
name in the status of the aggregated `ServiceImport`, which indicates the remote cluster was upgraded. Once the local
copies of all remote cluster `ServiceImports` have been removed, the local cluster's `ServiceImport` can be removed from
the broker.

The CoreDNS plugin still needs to watch for `ServiceImports` from non-upgraded clusters as well as the aggregated
`ServiceImport` to maintain continuity during upgrade. If present, a cluster's `ServiceImport` will be used instead of
its `EndpointSlice` when servicing requests to preserve prior behavior.

## Alternative Approaches

We also considered the alternative approaches described below.

### Active Broker

We considered adding an mcs-controller to the broker that would implement the reconciliation
logic from `ServiceExport` to `ServiceImport` resources. The goal was to have a clear separation
of logic for MCS resources:

- Clusters would sync resources to and from the broker
- The mcs-controller on the broker would reconcile `ServiceImport` object state from the
  provided `ServiceExport` + `Service` state.

This approach was rejected because it requires a controller to run on the broker. It is
preferred for the Submariner architecture to keep a passive broker.

### Merge at Importing Cluster

We considered merging `ServiceImport` objects at each importing cluster. This could be as simple
as syncing only the oldest `ServiceImport` to the cluster. However it is not possible to handle
`ServiceExport` conflicts at the importing clusters so new logic is required at the exporting
clusters anyway.

[^1]: [KEP-1645: Multi-Cluster Services API](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api#constraints-and-conflict-resolution)
