# Add support for 100% MCS compliance

Related issue:
[Add support for 100% MCS Compliance](https://github.com/submariner-io/enhancements/issues/229)

## Summary

Current Lighthouse implementation differs from MCS API [^1] on two key points for ClusterIP Services:

1. There is no VirtualIP [VIP] assigned in `ServiceImport`.
2. `EndpointSlices` do not contain individual PodIPs.

To better align with MCS API Lighthouse needs an option to allocate and track VirtualIPs in `ServiceImports` and use
PodIPs in `EndpointSlices`.

## Proposal

When creating `ServiceExport`, users will have the option to add an annotation for enabling VIP for that Service.
This annotation will also be added to the aggregated `ServiceImport`.

A configuration flag will also be added to the `Submariner` and `ServiceDiscovery` CRDs, and `subctl` to set the
options at the global level. This is for deployments where the user wants the option set for all exported Services and
doesn't want to set the annotation explicitly for each `ServiceExport`.

Default behavior with nothing configured would be same as it is currently i.e. no VIP and no PodIPs.

Global flag can't be changed at runtime and requires a reinstall of whole clusterset. Change of flag in CRs will require
a restart of Submariner on that cluster as it changes the behavior of `ServiceExports` without annotation.

### VIP CIDR and allocation

1. Default CIDR of `243.0.0/8` will be used for Clusterset. User can change this at the time of deploying broker.
2. Each Cluster will get a default subset CIDR of `243.0.0.0/16` for allocating VIPs to Services exported from that
cluster. User can change this at the time of join but this CIDR must be an unallocated subset of global VIP CIDR.
3. VIP allocation is done by the first cluster to export the `Service`.
4. When Service is exported on subsequent clusters, no VIP allocation is done if VIP already present on `ServiceImport`.
5. VIP is deallocated only when `ServiceImport` is deleted i.e. `Service` or `ServiceExport` are no longer present on any
of the clusters.
6. If Submariner is uninstalled on the cluster that allocated a given VIP, VIP is not changed on `ServiceImport`.
7. When `lighthouse-agent` starts, it should check `ServiceImport`s for any VIPs allocated from its VIP CIDR. Any such
VIPs should be added to its local VIP allocation cache. This will also help with scenarios where cluster that allocated
VIP has uninstalled Submariner and another cluster got the same VIP CIDR on installation.

Note that `1` and `2` above are similar to Globalnet CIDR.

### DNS Resolution

Lighthouse DNS will return VIP for `ServiceImports` if available. It will be up to any other solution external to
Submariner to load balance this VIP to PodIPs in `EndpointSlices`. If VIP is not present, it will behave as it does
today, i.e. return one of the ClusterIPs from `EndpointSlices`.

### Conflict Resolution

In case of conflict in VIP setting when exporting `Service`, configuration on the oldest `ServiceExport`
will be used. The `ServiceExport` in conflict will be exported with existing behavior and `ServiceExportConflict`
condition will be set. The `Valid` condition will also be set to `true` as Service itself is exportable.

### Creating `ServiceExport`

Here are the detailed steps when creating a `ServiceExport`:

1. When a new `ServiceExport` is created, check if enable VIP annotation is present. If not present, use global flag to
determine if need to use VIP or not.
2. If VIP is enabled based on condition above, check if `ServiceImport` exists or not.
3. If `ServiceImport` doesn't exist, allocate the VIP and add it to new `ServiceImport`. Add an annotation
`lighthouse.submariner.io/clusterset-ip-allocated-by=<cluster-id>` to identify the cluster that allocated the VIP.
4. If a `ServiceImport` exists, compare the annotation on the `ServiceExport` and global flag against the annotation
in `ServiceImport` for any conflicts.
5. In case of conflict, set `ServiceExportConflict` condition on the `ServiceExport` and continue with export.
6. Create `EndpointSlices` with PodIPs instead of ClusterIP. Follow existing rules for merging port information in
`EndpointSlices`.

### Updating annotation on ServiceExport

Changing annotation will not modify the current behavior. It will only change the conflict
condition on `ServiceExport` if it adds or resolves the conflict, depending on the change. If the users wish to change
the behavior for a given `Service`, they will need to delete all `ServiceExports` and recreate them with required
annotations.

Same behavior will also apply on restart after changing global flag.

### Deleting `ServiceExport`

Here are the detailed steps when deleting a `ServiceExport`:

1. Check the cluster list on `ServiceImport` to determine if this is the last cluster to Export the service.
2. If this is last cluster, deallocate the VIP if it is same CIDR as current cluster and delete the `ServiceImport`.
3. If not, proceed as usual irrespective of VIP annotation. This means deleting EndpointSlice, recalculating service
ports and updating cluster list on `ServiceImport`.

### Globalnet

Globalnet will not be supported yet. Supporting Globalnet with PodIPs will be a significant scale issue as each Pod
backing the Service will require a GlobalIngressIP.

For any use cases that require Globalnet support with this feature, a separate Enhancement can be proposed in the
future.

### Migration

Since there is no change in default behavior, nothing should be required for migration.

If users want to change VIP behavior on existing ServiceExports post deployment, they will need to delete all
`ServiceExports`.

### Restart

On restart, every lighthouse-agent should check aggregated `ServiceImport` for IPs allocated from its CIDR and allocate
them in local allocation cache.

## Design Details

### Global Flags for deploy broker

When deploying broker, `--enable-clusterset-ip` flag will be added. There will also be an `--clusterset-ip-cidr-range`
option to set the VIP CIDR for clusterset to a non-default option. Irrespective of `--enable-clusterset-vip` setting,
Clusterset CIDR Range will be set to default of `243.0.0.0/8` unless set to a different value. This is to support use
cases where global flag is not set but user configures VIP on individual `ServiceExport`.

Similar to Globalnet, this information will also be stored in `broker-info.subm` file and BrokerInfo ConfigMap.

### Flags for subctl join

Following flags will be added to `subctl join`:

* `enable-clusterset-ip` - set the default behavior to enable VIP on this cluster. If not specified, it will pick the
default from global configuration done during deploy broker.
* `--clusterset-ip-cidr=a.b.c.d/x` - set the VIP CIDR for this cluster. Default will be allocated from Clusterset VIP
CIDR Range confiured during deploy broker step.

### New CRD flags

Following CRDs will need to be modified to support these new flags

```Go
type BrokerSpec struct {
    ClustersetIPEnabled bool `json:"clustersetIPEnabled,omitempty"`
    ClustersetIPCIDRRange []string `json:"clustersetIPCIDRRange,omitempty"`
}
```

```Go
type SubmarinerSpec struct {
    ClustersetIPEnabled bool `json:"clustersetIPEnabled,omitempty"`
    ClustersetIPCIDR []string `json:"clusterserIpCidr,omitempty"`
}
```

```Go
type SubmarinerDiscoverySpec struct {
    ClustersetIPEnabled bool `json:"clustersetIPEnabled,omitempty"`
    ClustersetIPCIDR []string `json:"clusterserIpCidr,omitempty"`
}
```

### Annotations for `ServiceExport`

* `lighthouse.submariner.io/use-clusterset-ip` - Use VIP for the `ServiceExport`

### Pros

1. Doesn't modify existing behaviour
2. Allows users to select behavior on a per Service basis.

### Cons

1. More configuration options for users to consider.
2. More chances of `ServiceExports` on different clusters being in conflict due to wrong annotations etc.

### Backward Compatibility

None.

### Alternatives

* Only use Global flag for entire deployment. This was discarded in favor of annotations based approach to provide
flexibility to users as they can mix and match services with and without VIP.

## User Impact

Existing users will not be impacted in any ways. Users who wish to use this feature will need to reinstall Submariner
with new flags or use annotations on `ServiceExports`.

## References

[^1]: [KEP-1645: Multi-Cluster Services API](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api#constraints-and-conflict-resolution)
