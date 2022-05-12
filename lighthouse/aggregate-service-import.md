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

There are changes required in Lighthouse's handling of `EndpointSlices` before single
`ServiceImports` can be introduced.

- The `ServiceImports` for an exported service each contain the service VIP from the exporting
  cluster. `ServiceImports` cannot be merged like this because a `ServiceImport` cannot have
  more than one IP. The semantics of the IP are that it should be a cluster local Service VIP,
  if needed. The service VIPs from the exporting clusters should be stored in `EndpointSlices`,
  with a separate `EndpointSlice` per exporting cluster.

- The current GlobalNet implementation exports `EndpointSlices` for normal services that contain
  pod IP addresses from the originating cluster. The `EndpointSlice` purpose will need to change
  to contain the Globalnet service VIP for the originating cluster.

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
subsequent exporting clusters to reuse it, assuming they have matching port lists.

1. First exporting cluster creates a `ServiceImport` on the broker.
1. Subsequent exporting clusters reuse the existing `ServiceImport`, updating the status to
   include the id of the exporting cluster.
1. Exporting clusters create and sync `EndpointSlices` to the broker with the service VIP and a
   `multicluster.kubernetes.io/service-name` label that matches the `ServiceImport`.
1. All clusters sync the `ServiceImport` and `EndpointSlices` from the broker.

Submariner will take a strict approach to service equivalence where the namespace, name and port
lists must match in order for exported services to be considered equivalent. This is stricter
than the MCS specification and is discussed further below in "Deviation from MCS". A
`ServiceExport` with the same namespace and name as an existing `ServiceImport` but which does
not meet the strict equivalence critera will have its status set to conflict.

### DNS Resolution

Lighthouse currently implements DNS resolution by round-robin load balancing between the
`ServiceImports` that match the namespace/name pair. With a merged `ServiceImport`, the
per-cluster service VIPs will be stored on the `EndpointSlices`. DNS resolution will select from
the `EndpointSlices` associated with the `ServiceImport` instead of using `ServiceImports`
directly.

DNS Resolution behaviour for headless services will remain unchanged.

### Service VIPs in Importing Clusters

Given the way that Submariner operates, there is no need to assign a Service VIP to a
`ServiceImport` in the importing clusters. This means that `ServiceImports` can always have an
empty IP address attribute, like they do for headless services today.

In future it would be possible to allocate a Service VIP to each `ServiceImport` and implement
load balancing across the exporting clusters. This would allow more sophisticated load balancing
than is possible with the DNS resolver.

### Adding a ServiceExport

Here are the detailed steps for adding a `ServiceExport` to a cluster:

1. When a new `ServiceExport` is created, lighthouse checks if a `ServiceImport` already exists.
1. If not, a new `ServiceImport` is created based on the `Service` definition.
1. If present, lighthouse checks if the `Service` definition conflicts with the `ServiceImport`.
1. If in conflict, lighthouse sets the `ServiceExport` status to conflict.
1. If not in conflict, lighthouse updates `ServiceImport.Status.Clusters` to include the cluster
   name.
1. A new `EndpointSlice` is created for the service VIP (or Pod VIPs for headless) and synced to
   the broker.

### Removing a ServiceExport

Here are the steps for removing a `ServiceExport` from a cluster:

1. When a `ServiceExport` is removed, the associated `EndpointSlice` should be deleted.
1. Lighthouse updates `ServiceImport.Status.Clusters` to remove the cluster name.
1. If the `ServiceImport` status has an empty cluster list it should be removed.
1. Clusters with `ServiceExports` in conflict should watch for `ServiceImport` removal and
   create a new `ServiceImport` based on their local `Service` definition.

### EndopointSlice Behaviour

The `EndpointSlice` that gets created by an exporting cluster will contain one of:

- The service VIP (or Globalnet service VIP) for ClusterIP services
- The list of pod IPs (or Globalnet pod IPs) for Headless services

When an exporting cluster has no pods backing an exported service, the EndpointSlice for the
cluster will be withdrawn. This is to ensure that requests do not get directed to clusters that
have no pods to service the request.

## Deviation From MCS

Looking at the `ServiceImport` merging in more detail, the MCS specification asks for all port
definitions from the `ServiceExports` to be combined in the merged `ServiceImport`. This means
that an exporting cluster would need to merge its port definitions into the `ServiceImport`.
Conversely, when removing a `ServiceExport` any merged port definitions would need to be
removed. However an exporting cluster only has visibility of its local `ServiceExport` so cannot
unmerge with the information available locally.

The notion that services are equivalent when they have matching name and namespace but different
port lists seems problematic. It is an open question what behaviour is expected when a
connection request for service port is directed to a cluster that does not expose that port. It
is preferable to set `ServiceExport` status to conflict for any differences between the exported
services.

By choosing to be strict about service equivalence, we can avoid the following issues:

- Performing any port list merging or unmerging on the `ServiceImport`.
- Preventing a request from getting forwarded to a cluster that does not serve a specific port.

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
