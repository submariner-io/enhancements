# ClusterSetNetworkPolicy: An Approach to Multi-Cluster Network Policy

Epic: [Multi cluster network policies](https://github.com/submariner-io/enhancements/issues/176)

## Summary

Kubernetes provides a mechanism for policing network access to pods - the [NetworkPolicy].
The default mechanism allows specifying ingress and egress traffic rules for a set of peer pods (determined by a selector).
A user could define network policies to allow traffic across the clusterset, by specifying peers using "ip blocks",
but it would be impossible to use more abstract selectors to identify peers, as each cluster can only account for its own resources.

We propose a new `ClusterSetNetworkPolicy` CRD for specifying multi cluster network policies.
The CRD will have the same structure and fields as the default Kubernetes `NetworkPolicy`,
with additional fields that are relevant to the clusterset.
Each such policy will be translated by the local CoastGuard controller to one (or more) `NetworkPolicy` capable of allowing remote traffic.
These local policies will be updated by the controller in response to any change in the original `ClusterSetNetworkPolicy` or any remote
pod or namespace that are relevant to the policy.

## Proposal

### Problem Definition

Kubernetes provides a mechanism for policing network access to pods - the [NetworkPolicy].
The default mechanism allows specifying ingress and egress traffic rules for a set of peer pods (determined by a selector).
The source and destination peer can be either simple IP CIDRs, or a more user friendly abstraction which allows specifying pods or
namespaces via selectors.
This abstraction allows the users to focus on the "what" instead of the "how", and leaves the implementation details to the CNI.

Such policies work well (when supported by the CNI) for a single cluster's network domain,
but what happens when the cluster is part of a clusterset?

When deploying in a clusterset, the network is essentially "stretched" across the clusterset,
providing a broader network domain than that of each single cluster.
We distinguish between network traffic going out of one cluster entirely (north-south),
and traffic traveling across the clusterset (east-west).
In such a scenario, east-west network traffic could be considered "internal traffic".
A user could define network policies to allow such traffic in the clusterset, by specifying peers using "ip blocks",
but it would be impossible to use more abstract selectors to identify peers, as each cluster can only account for its own resources.

What we should strive to solve, then, is the ability for users to define network policies for east-west traffic in a clusterset,
while allowing them to use higher level abstractions for defining the policy peers.

### Design Limitations

#### CNI Agnosticity

Submariner aims to be as CNI agnostic as possible, and as such tries to interfere as little as possible with the underlying CNI.
As such, we have a limited capacity to transmute the network traffic (e.g. we can't plant any additional metadata inside a packet).

#### NetworkPolicy Interoperability

As a clusterset implementation, we would still need to be aware of the possibility of localized single-cluster network policies.
As such, we should not interfere with any such policies, should they be defined.
Since these policies could inherently disallow inter-cluster traffic,
our design and implementation should strive to complement them by adding support for such traffic.

#### Pod IP Preservation

For all practical applications of this solution, we would need to preserve the source IP in the packet 
either by avoiding any NAT on the source cluster, or by utilizing a solution such as an overlay network.
We would need to understand how this works with Globalnet v2 (if it's at all possible).

### Design Tenets

#### Broker Centric Data Sharing

As we're running in the Submariner ecosystem, it makes sense to share any data we need to share via the broker.
This has the following benefits (and more that aren't listed):

* Debuggability - users can see and query the data on the broker with ease to trace any problems.
* Standardized clients - we have standardized Kubernetes clients generated, with all the common features (selectors, etc).
* Integrated RBAC and authentication - Kubernetes takes care of this aspect for us as well.

#### Connectivity Decoupling

We should strive to decouple the design and implementation from the underlying connectivity,
building on top of the fact that connectivity exists but not counting on it to be implemented in some certain way.
This would help our solution to be more robust and versatile.

#### Solution Scope

* We aim to solve the problem of having clusterset-aware network policies.
* We're not trying to provide a revolutionary approach,
  but rather an evolutionary approach that expands the base Kubernetes network policy concept.
* We aim for a good enough solution, but not a perfect solution encompassing every possible use case and edge case.

## Design Details

### ClusterSetNetworkPolicy CRD

A new `ClusterSetNetworkPolicy` CRD will be defined for specifying multi cluster network policies.
The CRD will be namespace scoped (same as the Kubernetes NetworkPolicy resource).
The CRD will have the same structure and fields as the default Kubernetes [NetworkPolicy],
with additional fields that are relevant to the clusterset.

The policies are local to the cluster, meaning they affect only the pods they are targeting on that cluster.
If users want, they can create the same policy across different clusters manually.
In the future, we may consider making policies global (or adding such a capability via a new CRD),
allowing the users to create the policy once, and the system will take care of distributing it.

Each such policy will be translated by the local CoastGuard controller to one (or more) `NetworkPolicy` where:

* Local peers use the selector defined in the `ClusterSetNetworkPolicy` CR.
* Remote peers (pods) are identified using an `ipBlock` with a `/32` CIDR.

These local policies will be updated by the controller in response to any change in the original `ClusterSetNetworkPolicy` or any remote
pod or namespace that are relevant to the policy.

#### Proposed CR Structure

```yaml
apiVersion: submariner.io/v1alpha1
kind: ClusterSetNetworkPolicy
metadata:
  name: <string>
  namespace: <string>
spec:
  podSelector: <LabelSelector>
  policyTypes: <string array>
  ingress:
  - <NetworkPolicyIngressRule>
  - from:
    - <NetworkPolicyPeer>
    - namespaceSelector: <LabelSelector>
      podSelector: <LabelSelector>
    ports: <NetworkPolicyPort array>
  egress:
  - <NetworkPolicyEgressRule>
  - to:
    - <NetworkPolicyPeer>
    - namespaceSelector: <LabelSelector>
      podSelector: <LabelSelector>
    ports: <NetworkPolicyPort array>
```

### Tracking Remote Pod IPs

#### RemotePod & RemoteNamespace CRDs

A simple approach to tracking remote pods is having a specific CRD representing each remote pod and a CRD representing remote namespaces.
These CRDs would be namespace-scoped but the namespace it's in will be a system set namespace (e.g. submariner-operator).
These CRDs will be synchronized via the broker.

Each cluster will track its own pods and namespaces and keep them updated on each change in their lifecycle:

* When a pod/namespace is created, create a corresponding `RemotePod`/`RemoteNamespace`.
* When a pod/namespace is updated, update the corresponding `RemotePod`/`RemoteNamespace`.
* When a pod/namespace is deleted, delete the corresponding `RemotePod`/`RemoteNamespace`.

The advantage of this approach is that it's quite simple for each cluster to track and generate these CRs,
and each consuming cluster will use them for the actual implementation.
A disadvantage of this approach is that we'll be tracking every pod on every cluster, even if they don't match any selector.
This can be mitigated somewhat by having the source cluster track only pods/namespaces that match relevant selectors.

##### Proposed RemotePod CR Structure

```yaml
apiVersion: submariner.io/v1alpha1
kind: RemotePod
metadata:
  labels: <map string -> string>
  name: <string>
  namespace: <string>
spec:
  clusterID: <string>
  podIPs: <list of strings>
  remoteNamespace: <string>
```

##### Proposed RemoteNamespace CR Structure

```yaml
apiVersion: submariner.io/v1alpha1
kind: RemoteNamespace
metadata:
  labels: <map string -> string>
  name: <string>
  namespace: <string>
spec:
  clusterID: <string>
```

### Pros

The advantages of this proposal are:

* A similar CRD structure would be more familiar to existing Kubernetes users.
* Translating the structure to vanilla Kubernetes `NetworkPolicy` objects should be straightforward.
* Not interfering with any existing vanilla `NetworkPolicy` resources.
* The cluster's CNI takes care of the actual implementation details of the policy enforcement.
* Under complete control of Submariner, rather than risking conflicts with other systems that might be managing these resources.
* Ability to define additional clusterset-wide concerns in the CRD.

### Cons

Most of the cons concern possible scale and/or performance issues:

* We could face scale/performance issues with updating the broker on every change of a pod, which could overwhelm the entire API.
* Maintaining translated `NetworkPolicy` objects with many `ipBlock` peers could face scale/performance issues.

We would have to implement a POC to be able to assess the possibility of hitting these issues, and any possible mitigations.

The benefits of relying on existing Kubernetes capabilities generally outweigh the possible risks.
However, in the future we could switch to a more scalable implementation while retaining the core `ClusterSetNetworkPolicy` API.

### Backward Compatibility

There aren't any concerns, as this is a new API with new CRDs.

### Alternatives

#### Aggregate Using a PolicyPeerPodIPs CRD

For each of the `NetworkPolicyPeer` definitions, we would need to collect the pod IPs that match the selectors on each of the clusters.
For this purpose, we will use a separate CRD that represents these IPs on each given cluster.

As with the `ClusterSetNetworkPolicy` CRD, this CRD is namespace scoped (to match with its `ClusterSetNetworkPolicy`).
The CRD is synchronized across all clusters using the broker.
Each cluster will maintain and update only its own list.
Each resource will be linked to the cluster it represents, and to the `ClusterSetNetworkPolicy` it appears in.

```yaml
apiVersion: submariner.io/v1alpha1
kind: PolicyPeerPodIPs
metadata:
  name: <string>
  namespace: <string>
spec:
  clusterID: <string>
  clusterNetworkPolicy: <string>
  clusterSelector: <LabelSelector>
  namespaceSelector: <LabelSelector>
  podIPs:
    - <string>
  podSelector: <LabelSelector>
```

This approach is much more complicated as each source cluster will have to track and compose this information for each of the policies,
but it could reduce API call overhead (although the number of API calls overall is expected to be the same).
We opt for the simpler approach for now, but could consider this approach if we see that it might alleviate scale/performance issues.

#### Peer-to-peer Sharing of Data

This approach would use an internal protocol to exchange information P2P relying on the existing IP connectivity/via service exports.
Every network policy controller would declare its existence to other clusters via `ServiceExports`.
All the controllers would connect to each other exchanging the pod selectors of their interest,
and receiving updates based on those selectors.

The benefits of this alternative are:

* Adds no pressure to the broker API/etcd.
* Adds no pressure in the form of replicated objects in the local APIs/etcds.

The disadvantages of this alternative are:

* A way to provide mutual authentication is necessary (mTLS or equivalent,
  something needs to provide certificates to the different controllers in a way that they trust each other).
* It needs a protocol to be designed. We believe there are solutions to do this type of communication in Kubernetes,
  but not sure if it could be based on protobuf or something else.
* Possibly less visible, but could be handled with good logging.
* Generally more code to maintain rather than reusing existing capabilities.
* It deviates from the standard Submariner broker-centric architecture used in other projects.

## External Dependencies

None

## User Impact

Users should be able to define network policies that are aware of clusterset-wide traffic.

[NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
