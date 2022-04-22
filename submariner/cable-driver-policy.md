# Cable Driver Policy Support

## Summary

Submariner currently offers three cable drivers: VXLAN, IPSec or wireguard. However only one of these cable drivers
is enabled across all the clusters in a ClusterSet; That is all clusters that join a broker must use the same cable
driver. The goal of this proposal is to extend the current model to support the selection of different cable types
between connected clusters.

There are use cases that involve supporting a mix of cable drivers or cable driver configurations within the same
ClusterSet. For example, a ClusterSet could contain three clusters, two that are on-premise and the other
in a public cloud. Connections to the cloud cluster will need to use encryption while the on-premise clusters can use
unencrypted connections, as they’re connected by a private link, to avoid the performance overhead of encryption.

## Proposal

This enhancement proposes a new policy CRD to allow users to selectively deploy different cable types between clusters
in a ClusterSet. The proposal assumes that connection between two clusters is symmetrical. If Cluster A is connected
to Cluster B via a VXLAN cable type - then Cluster B is connected to Cluster A via a VXLAN cable type and not another
cable type.

This enhancement will propose a single API for cable driver policy support using custom resource definition (CRDs).

### Cable driver policy API

This following CRD is proposed:

```Go
type ClusterConnectionPolicy struct {
    metav1.TypeMeta `json:",inline"`

    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec ClusterConnectionPolicySpec `json:"spec"`
}

type ClusterConnectionPolicySpec struct {
    // LeftClusterSelector identifies the Cluster resources representing the clusters on one end of a connection. An empty
    // selector indicates wildcard, ie matches any cluster.
    LeftClusterSelector metav1.LabelSelector `json:"leftClusterSelector,omitempty"`

    // RightClusterSelector identifies the Cluster resources representing the clusters on the other end of a connection.
    // An empty selector indicates wildcard, ie matches any cluster.
    RightClusterSelector metav1.LabelSelector `json:"rightClusterSelector,omitempty"`

    // Name of the cable driver implementation to use for this connection.
    CableDriver string `json:"CableDriver"`

    // +optional
    // NoConnectOnConflict defines the intention to fallback to no connection if there's a policy conflict
    // If defined - should be part of the default policy
    NoConnectOnConflict bool `json:"NoConnectOnConflict"`
}
```

The `ClusterConnectionPolicy` CR defines the criteria for selecting which cable driver to use to connect cluster pairs (or
subsets). To resolve the connectivity policy for a remote cluster's Endpoint, Submariner would:

- Search for a policy whose left and right label selectors match the corresponding Cluster resources of the local and remote
  Endpoints.

> **_NOTE:_** In the case of multiple labels being specified, the selector will need to match all the labels.
> Inequality-based requirements will be supported. For e.g. labels such as `env: !production` will match any clusters
> that aren't labelled with `env: production`.

If any matching policy is found, the clusters are connected using the cable driver specified in the cable driver policy.

If no matching policy is found, the `default` cable driver policy is used. The `default` policy can be auto-created at broker
deploy time.

If more than one matching policy is found, the default policy will consulted to establish if `NoConnectOnConflict` is set to true.
In the case where `NoConnectOnConflict` is true the clusters are **not connected** and the connection status between a pair of clusters
is marked as `conflict`. In the case where `NoConnectOnConflict` is false, the clusters are connected but the connection status between
a pair of clusters is marked as `conflict`. If `NoConnectOnConflict` is undefined, it's assumed to be set to True.

The initial `ClusterConnectionPolicy` resources would be created and maintained on the broker cluster and synced to each cluster in the
ClusterSet.

Submariners `EndpointSpec` needs to be modified as `Backend` and `BackendConfig` only reflect a single cable driver. An alternative parameter
`Backends` is proposed. This parameter represents a slice of supported backends and their configurations on an Endpoint. This information
can be used in conjunction with the `ClusterConnectionPolicy` to validate a connection can be established (i.e. both ends of the connection
support the same driver).

The `EndpointSpec` could also be modified to advertise which policies it supports to further support the search for a suitable policy for
a remote gateway trying to connect to that endpoint. The `SupportedPolicies` parameter is a simple slice of strings that identifies the set
of policies that a Gateway has deemed as being applicable to it from the set of `ClusterConnectionPolicy` CRs.

```go
type EndpointSpec struct {
  // +kubebuilder:validation:MaxLength=63
  // +kubebuilder:validation:MinLength=1
  ClusterID string `json:"cluster_id"`
  CableName string `json:"cable_name"`
  // +optional
  HealthCheckIP      string            `json:"healthCheckIP,omitempty"`
  Hostname           string            `json:"hostname"`
  Subnets            []string          `json:"subnets"`
  PrivateIP          string            `json:"private_ip"`
  PublicIP           string            `json:"public_ip"`
  NATEnabled         bool              `json:"nat_enabled"`
  Backends           []BackendSpec     `json:"backends,omitempty"`
  SupportedPolicies  []string          `json:"supported_policies,omitempty"`
}

type BackendSpec struct {
    CableDriver      CableDriver        `json:"cable_driver,omitempty"`
}

type CableDriver struct {
    Name       string                   `json:"name,omitempty"`
    Config     map[string]string        `json:"config,omitempty"`
}
```

The `GatewayStatus` should be updated to reflect the supported backends rather than just one.

```go
type GatewayStatus struct {
  Version       string       `json:"version"`
  HAStatus      HAStatus     `json:"haStatus"`
  LocalEndpoint EndpointSpec `json:"localEndpoint"`
  StatusFailure string       `json:"statusFailure"`
  Connections   []Connection `json:"connections"`
  Backends      []BackendSpec   `json:"backends"`
}
```

The `ConnectionStatus` that reflects the state of different connections on a Submariner gateway
should be extended to include a new message `conflict` for situations where conflicting policies
exists for a connection between clusters.

```go
type ConnectionStatus string

const (
  Connected          ConnectionStatus = "connected"
  Connecting         ConnectionStatus = "connecting"
  ConnectionError    ConnectionStatus = "error"
  ConnectionConflict ConnectionStatus = "conflict"
)
```

The Connection information stored on a Gateway should be updated to reflect what policy is in use for that connection.

```go
type Connection struct {
  Status        ConnectionStatus  `json:"status"`
  StatusMessage string            `json:"statusMessage"`
  Endpoint      EndpointSpec      `json:"endpoint"`
  UsingIP       string            `json:"usingIP,omitempty"`
  UsingNAT      bool              `json:"usingNAT,omitempty"`
  PolicyID      string            `json:"policy_id,omitempty"`
  cableType     string            `json:"cable_type,omitempty"`
  // +optional
  LatencyRTT *LatencyRTTSpec     `json:"latencyRTT,omitempty"`
}
```

### Cluster local Connections Resources

The detailed connection information for a Gateway is currently maintained in the Gateway and Submariner CRs,
however when the number of clusters grows to a large scale these could be difficult to parse. It maybe preferable
to maintain connection information in a separate set if `ClusterConnection` CRs. Maintaining this info in separate
set of CRs could be useful in the case of:

- Gateway failover, so the new gateway doesn't need to do the same work again to build a connection definition. Or
- In the case of `ClusterConnectionPolicy` updates. A Gateway could quickly cycle through the connection policies its
using in the `ClusterConnection` CRs to see if an update impacts it.

> **_NOTE:_** The scope of a `ClusterConnectionPolicy` CR is across the ClusterSet. The scope of a `ClusterConnection` is local to a cluster.

The proposed Connection CRD is shown below.

```go
type ClusterConnection struct {
    metav1.TypeMeta `json:",inline"`

    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec ClusterConnectionSpec `json:"spec"`
}

type ClusterConnectionSpec struct {
    // LocalCluster identifies the Cluster resource on one end of a connection.
    // Potentially not needed
    LocalCluster string `json:"local_cluster,omitempty"`

    // RemoteCluster identifies the Cluster resource on the other end of a connection.
    RemoteCluster string `json:"remote_cluster,omitempty"`

    // Name of the cable driver implementation used for this connection.
    CableDriver string `json:"cable_driver"`

    // Name of the policy used for this connection.
    // Could potentially be used as a label for a ClusterConnection
    Policy string `json:"policy"`

    // Status of the connection
    Status ClusterConnectionStatus `json:"status"`

    LocalEndpoint EndpointSpec `json:"local_endpoint"`

    RemoteEndpoint EndpointSpec `json:"remote_endpoint"`

    UsingIP  string `json:"usingIP,omitempty"`

    UsingNAT bool   `json:"usingNAT,omitempty"`

    // +optional
    LatencyRTT *LatencyRTTSpec     `json:"latencyRTT,omitempty"`
}

// ClusterConnectionStatus describe the current state of the ClusterConnectionPolicy.
type ClusterConnectionStatus struct {
  // Conditions holds an array of metav1.Condition that describe the state of the ClusterConnectionStatus.
  Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// ClusterConnectionPolicyConditionType is the type for status conditions on
// a ClusterConnectionPolicy This type should be used with the
// ClusterConnectionStatus.Conditions field.
type ClusterConnectionConditionType string

const (
  ClusterConnectionConnected ClusterConnectionConditionType = "Connected"
  ClusterConnectionNotConnected ClusterConnectionConditionType = "Not Connected"
  ClusterConnectionError ClusterConnectionConditionType = "Error"

)

//   ClusterConnectionConditionReason defines the set of reasons that explain why a
// particular ClusterConnectionPolicy condition type has been raised.
type   ClusterConnectionConditionReason string

const (
  // ClusterConnectionConditionReasonPolicyConflict represents a reason where the ClusterConnectionPolicy may not have been
  // implemented in the cluster due to a conflict detected with another ClusterConnectionPolicy
  ClusterConnectionConditionReasonPolicyConflict  ClusterConnectionConditionReason = "Policy Conflict"
)
```

> **_NOTE:_** The `ClusterConnection` CR could be a single CR that aggregates the status of all connections.

### Labelling clusters

Clusters are defined as Custom Resources and can be labeled in the CRD. A new parameter to label clusters at
`subctl join` time could also be added for usability purposes.

#### Connection Policy Examples

- We want to specify a default policy that uses IPsec as the default cable driver.

```yaml
kind: ClusterConnectionPolicy
metadata:
  name: default
  namespace: submariner-operator
spec:
  cableDriver:
    name: "ipsec"
```

- We want to use specific cable drivers to connect clusters in a ClusterSet as shown in the image below. We
label the `Clusters` appropriately and define a number of policies with the appropriate label selectors.

![Finer grained connectivity example](./images/multiple_cables_ep.png)

```yaml
kind: ClusterConnectionPolicy
metadata:
  name: default
  namespace: submariner-operator
spec:
  cableDriver:
    name:  "vxlan"
```

```yaml
kind: ClusterConnectionPolicy
metadata:
  name: on-prem-to-cloud
  namespace: submariner-operator
spec:
  leftClusterSelector:
    env: cloud
  rightClusterSelector:
    env: on-premise
  cableDriver:
    name:  "ipsec"
```

#### Updated Endpoint example

```yaml
Kind:         Endpoint
Metadata:
  Name:         cluster1-submariner-cable-cluster1-172-18-0-14
  Namespace:    submariner-operator
Spec:
  Backends:
    cable_driver:
        name: vxlan
        config:
          Natt - Discovery - Port:  4490
          Preferred - Server:       false
          Udp - Port:               4789
    cable_driver:
        name: ipsec
        config:
          Natt - Discovery - Port:  4490
          Preferred - Server:       false
          Udp - Port:               4789
  cable_name:                 submariner-cable-cluster1-172-18-0-14
  cluster_id:                 cluster1
  Health Check IP:            10.1.192.0
  Hostname:                   cluster1-worker
  nat_enabled:                false
  private_ip:                 172.18.0.14
  public_ip:                  66.187.232.132
  Subnets:
    100.1.0.0/16
    10.1.0.0/16
```

### Submariner Cable Driver Configuration

It's expected that on Gateway startup, the `ClusterConnectionPolicy` CRs imported from the broker will be
parsed to determine which cable drivers the Submariner Gateway should initialize.
