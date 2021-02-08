# Fine Grained Connectivity

## Summary

Submariner currently provisions full mesh topology by default between the participating clusters.
For some use-cases, we need fine-grained connectivity options. This proposal covers only the topology
aspect of connectivity.

This proposal aims to address the below use-case

Central Cluster Topology -  Here only the client can talk to the server but not to the other clients to
which the server is connected to. The server can talk to all connected clients.

## Proposal

### Terminology

#### Clusterset

A Clusterset is a group of clusters that forms a trust boundary. The design of Clusterset can be aligned
to the ClusterID KEP. More details are available in the below link.

[Cluster-Id KEP]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/2149-clusterid

In this proposal, we intend to use only the ClusterId(which exists already) and ClusterSet as a part of
Submariner Cluster CRD. The ClusterClaim API shall be realized once the KEP is finalized upstream.

### Design Details

![Clustersets](./images/clustersets.png)

The Submariner should be enhanced to support grouping of clusters into Clustersets. The Cluster CRD can
be enhanced to hold the Clustersets name in which the cluster is a part of. The one or more Clustersets
the cluster wants to be a part of shall be specified when the cluster joins the broker. If no
Clustersets are specified, the default will be used. The cluster-id should be unique across Clustersets.

```Go

type Cluster struct {
        metav1.TypeMeta   `json:",inline"`
        metav1.ObjectMeta `json:"metadata,omitempty"`
        Spec              ClusterSpec `json:"spec"`
}

type ClusterSpec struct {
        ClusterID   string   `json:"cluster_id"` // perhaps this could just be a hash of the name...?
        ClusterSets []string `json:"cluster_sets"`
        ServiceCIDR []string `json:"service_cidr"`
        ClusterCIDR []string `json:"cluster_cidr"`
        GlobalCIDR  []string `json:"global_cidr"`
}
```

#### Submariner Broker

There will be a single broker running in one of the participating clusters. The clusters will specify
the Clusterset it wants to be a part of when it joins the broker. A cluster can be a part of multiple Clustersets.
The broker ensures that resources are only synced within a Clusterset.

#### Service Discovery

The broker ensures that the ServiceImports are synced only within the Clustersets. So a cluster should be a
part of the Clusterset from which the service was exported to receive the service import.  But the same
service in the same namespace across Clustersets would be considered an instance the same service.

#### Dataplane Connectivity

Similar to ServiceImports, endpoint objects are synced only within the Clustersets. A full mesh of tunnels
will be created within the members of the Clustersets. If a cluster is a part of two Clusterset multiple
tunnels will be created, but will not forward traffic from one Clusterset to other.

#### GlobalNet

Globalnet currently allocates CIDR per cluster. So across Clustersets each cluster will have non-overlapping CIDR.

## User Story

1) Central Cluster Topology: To achieve each server and client will be grouped into an isolated group of
clusters called Clusterset. A Server cluster which is part of multiple Clustersets does not perform routing between
the Spokes in different Clustersets. There will be a full mesh topology within the Clusterset, though in this use case it
will be just two clusters in the Clusterset. A cluster can be a part of multiple Clusterset. So the server can be a
part of multiple Clustersets, where it connects with a client in each Clusterset

## Pros

1) The use-cases can be achieved without compromising the submariner promise of full-mesh connectivity between
 all the connected clusters
clusters.
2) The changes are mostly restricted to the broker components and easier to implement.

## Cons

1) There may be an issue of scale if there are many 2 member cluster sets. The broker needs to do the extra work
to ensure that the resources are synced only to the respective clusters.
2)A cluster joining multiple Clusterset is defined as "out of scope" in the current cluster-id KEP. So allowing
it may affect future alignment with upstream KEP.

## Alternatives

One broker per Clusterset: With multiple brokers either we need multiple Submariner components each having its own
namespace and talk to a specific broker or we need to have a submariner component talking to multiple brokers.
Either option requires changes across Submariner and Lighthouse. A significant change will be required in the
submariner-operator and deployment as well.
