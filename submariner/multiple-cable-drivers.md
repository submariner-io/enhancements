# Support fine-grained cluster connectivity and multiple cable drivers

## Summary

Submariner currently assumes full mesh by default and provisions VPN connections between all participating clusters. For
some use cases, it makes sense to offer finer-grained connectivity options. One such use case is a client-server scenario
where multiple client clusters need to access a server cluster but the user does not want the client clusters to connect
to one another. Finer-grained connectivity could also improve overall scalability where full mesh is not required.

Submariner supports several cable driver implementations (libreswan, wireguard et al) however only one can be
configured within the same cluster set, that is all clusters that join a broker must use the same cable driver.

There are use cases that involve supporting a mix of cable drivers or cable driver configurations within the same
cluster set. For example, a cluster set could contain three clusters, two that are on-premise and the other
in the cloud. Connections to the cloud cluster will need to use encryption while the on-premise clusters can use
unencrypted connections, as they’re connected by a private link, to avoid the performance overhead of encryption.
This could be achieved using different cable driver implementations or the same cable driver implementation with
differing encryption configurations.

## Proposal

This enhancement proposes a mechanism to allow users to specify their intent as to which clusters to connect and how to
connect them. The default would still be a full mesh with an homogeneous cable driver.

The standard mechanism in Kubernetes to specify intent is by creating a Kubernetes API resource whose effect is eventually
realized into the cluster's desired state. To extend the Kubernetes API, one could either use the generic Kubernetes
`ConfigMap` or use a custom resource definition (CRD). There's pros and cons to each but a CRD seems more suitable for
this enhancement.

The finer-grained connectivity and multiple cable driver use cases could be enabled via the same API or separate ones.
Having one API may be a bit simpler in some respects but it's not foreseen that both use cases would overlap and they
could interfere with one another and yield undesired results if the user isn't careful as the semantics differ. Therefore
two APIs with some similar characteristics will be proposed.

### Finer-grained connectivity API

This following CRD is proposed:

```Go
type ClusterConnectivityPolicy struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec ClusterConnectivityPolicySpec `json:"spec"`
}

type ClusterConnectivityPolicySpec struct {
    // LeftClusterSelector identifies the Cluster resources representing the clusters on one end of a connection. An empty
    // selector indicates wildcard, ie matches any cluster.
    LeftClusterSelector metav1.LabelSelector `json:"leftClusterSelector,omitempty"`

    // RightClusterSelector identifies the Cluster resources representing the clusters on the other end of a connection.
    // An empty selector indicates wildcard, ie matches any cluster.
    RightClusterSelector metav1.LabelSelector `json:"rightClusterSelector,omitempty"`
}
```

The `ClusterConnectivityPolicy` CR defines the criteria for selecting which cluster pairs can connect. To resolve the
connectivity policy for a remote cluster's `Endpoint`, Submariner would search for a policy whose left and right label
selectors match the corresponding `Cluster` resources of the local and remote `Endpoints`. If any matching policy is found,
the clusters are connected, otherwise they are not.

If there are no policies defined, Submariner would default to full mesh connectivity.

`ClusterConnectivityPolicy` resources would be created and maintained on the broker cluster and synced to each participating cluster.

#### Examples

- We only want certain clusters to access a database server in another cluster and no other clusters to connect to one another. We
label the server and client `Clusters` appropriately and define a single policy with corresponding label selectors. Since there's
no other policy to match client cluster pairs, they will not be connected.

```yaml
apiVersion: v1
kind: ClusterConnectivityPolicy
metadata:
  name: client-server-policy
spec:
  leftClusterSelector:
    database-role: server
  rightClusterSelector:
    database-role: client
```

- We want any cluster to access a database server in another cluster but no other clusters to connect. We label the server
`Cluster` appropriately and define a single policy with the corresponding label selector and an empty selector to match all.

```yaml
apiVersion: v1
kind: ClusterConnectivityPolicy
metadata:
  name: client-server-policy
spec:
  leftClusterSelector:
    database-server: true
  rightClusterSelector:
```

### Multiple cable drivers API

This following CRD is proposed:

```Go
type ClusterConnectionPolicies struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec ClusterConnectionPoliciesSpec `json:"spec"`
}

type ClusterConnectionPoliciesSpec struct {
    Policies []ClusterConnectionPolicy `json:"policies,omitempty"`
}

type ClusterConnectionPolicy struct {
    // LeftClusterSelector identifies the Cluster resources representing the clusters on one end of a connection. An empty
    // selector indicates wildcard, ie matches any cluster.
    LeftClusterSelector metav1.LabelSelector `json:"leftClusterSelector,omitempty"`

    // RightClusterSelector identifies the Cluster resources representing the clusters on the other end of a connection.
    // An empty selector indicates wildcard, ie matches any cluster.
    RightClusterSelector metav1.LabelSelector `json:"rightClusterSelector,omitempty"`

    CableDriver *CableDriverSpec `json:"cableDriver,omitempty"`
}

type CableDriverSpec struct {
    // Name of the cable driver implementation.
    Name string `json:"name"`

    // Options specifies key/value configuration parameters for the cable driver.
    Options map[string]string `json:"options,omitempty"`
}
```

The `ClusterConnectionPolicies` CR defines a list of policies that are applied in order of precedence to select the cable driver
and associated optional configuration options to use for each cluster connection. To resolve the connection policy for a remote
cluster's `Endpoint`, Submariner would iterate the list of policies to search for a policy whose left and right label selectors
match the corresponding `Cluster` resources of the local and remote `Endpoints`. The first matching policy is used. If no matching
policy is found, the default cable driver is used.

Defining the policies as an ordered list allows the user to resolve ambiguity should a cluster pair match multiple policies.
Otherwise, if Submariner encountered such a case, it would either have to arbitrarily pick a policy or reject all of them and
use the default. Neither option is ideal. An alternative to the ordered list would be to have separate resources for each policy
and define a priority or precedence field. However it seems a single resource without the use of magic priority numbers would be
easier for the user to maintain.

The `ClusterConnectionPolicies` CR would be a singleton with a well-known name and would be maintained on the broker cluster
and synced to each participating cluster.

#### Examples

- We want to use a specific cable driver to connect on-premise clusters. We define a policy to select clusters labeled as
`on-premise` to use a `vxlan` cable driver. The default cable driver will be used to connect everything else.

```yaml
apiVersion: v1
kind: ClusterConnectionPolicies
metadata:
  name: cluster-connection-policies
spec:
  policies:
    - leftClusterSelector:
        location: on-premise
      rightClusterSelector:
        location: on-premise
      cableDriver:
        name: vxlan
```

- We want to use a specific cable driver to connect on-premise clusters and another cable driver to connect everything else.
We define a policy to select clusters labeled as `on-premise` to use a `vxlan` cable driver and a fall-back policy that selects
all other clusters to use `wireguard`.

```yaml
apiVersion: v1
kind: ClusterConnectionPolicies
metadata:
  name: cluster-connection-policies
spec:
  policies:
    - leftClusterSelector:
        location: on-premise
      rightClusterSelector:
        location: on-premise
      cableDriver:
        name: vxlan
    - leftClusterSelector:
      rightClusterSelector:
      cableDriver:
        name: wireguard
```
