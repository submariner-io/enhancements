# Support fine-grained cluster connectivity and multiple cable drivers

## Summary

Submariner currently assumes full mesh connectivity in a ClusterSet by default and provisions VPN connections
between all participating clusters. In addition, Submariner currently offers one of three cable drivers: VXLAN,
IPSec or wireguard. Only one of these cable drivers is enabled across all the clusters in a ClusterSet, that is
all clusters that join a broker must use the same cable driver. The goal of this proposal is to outline an extended
model that will support:

- The selection of which clusters in a ClusterSet to connect/not connect.
- The selection of one or more cable drivers in order to support different cable types between connected clusters,
  and how to configure them. For example, an IPSec cable between public and private clouds but a VXLAN cable between
  the clusters in the same cloud.

There are use cases that involve supporting a mix of cable drivers or cable driver configurations within the same
ClusterSet. For example, a ClusterSet could contain three clusters, two that are on-premise and the other
in a public cloud. Connections to the cloud cluster will need to use encryption while the on-premise clusters can use
unencrypted connections, as they’re connected by a private link, to avoid the performance overhead of encryption.
This could be achieved using either:

- Different cable driver implementations or
- The same cable driver implementation with differing configurations.

For other use cases, it makes sense to offer finer-grained connectivity options. One such use case is a scenario
where multiple clusters need to access a service offered by another cluster but the user does not want to connect
the clusters that require the service to one another. Finer-grained connectivity could also improve overall
scalability where full mesh is not required.

## Proposal

This enhancement proposes a mechanism to allow users to specify what clusters can connect and what cable
driver to use to connect them. The default would still be a full mesh connectivity policy with a homogeneous
cable driver.

The idea is that the user will be able to selectively deploy different cable types between clusters in
a ClusterSet. They will be able to select a VXLAN cable type between clusters co-located in the same
cloud (assuming they are in the same ClusterSet) as well as an IPSec cable between clusters in different
clouds.

In addition to supporting varying cable types between clusters in a ClusterSet, the user will also select
which clusters to connect or not connect.

> **_NOTE:_** The proposal assumes that connection between two clusters is symmetrical. If Cluster A is
connected to Cluster B via a VXLAN cable type - then Cluster B is connected to Cluster A via a VXLAN cable
type and not another cable type.

The standard mechanism in Kubernetes to specify intent is by creating a Kubernetes API resource whose effect
is eventually realized into the cluster's desired state. To extend the Kubernetes API, one could either use
the generic Kubernetes `ConfigMap` or use a custom resource definition (CRD). There's pros and cons to each
but a CRD seems more suitable for this enhancement.

This enhancement will propose a single API for finer-grained connectivity and multiple cable driver support.

### Finer-grained connectivity and multiple cable driver API

This following CRD is proposed:

```Go
type ClusterConnectionPolicies struct {
    metav1.TypeMeta `json:",inline"`

    metav1.ObjectMeta `json:"metadata,omitempty"`

    ConnectionPolicies []ClusterConnectionPolicySpec `json:"spec"`
}

type ClusterConnectionPolicySpec struct {
    // LeftClusterSelector identifies the Cluster resources representing the clusters on one end of a connection. An empty
    // selector indicates wildcard, ie matches any cluster.
    // Not used in combination with ClusterSelector.
    LeftClusterSelector metav1.LabelSelector `json:"leftClusterSelector,omitempty"`

    // RightClusterSelector identifies the Cluster resources representing the clusters on the other end of a connection.
    // An empty selector indicates wildcard, ie matches any cluster.
    // Not used in combination with ClusterSelector.
    RightClusterSelector metav1.LabelSelector `json:"rightClusterSelector,omitempty"`

   // Priority identifies the priority for the connection policy. Higher priorities are applied first.
    Priority  *int  `json:"priority,omitempty"`

    // ConnectionPolicy identifies the default connection policy if no policies are found: to connect or no-connect.
    ConnectionPolicy *string `json:"connectionPolicy,omitempty"`

    CableDriverConfigs []CableDriverSpec `json:"spec"`
}

type CableDriverSpec struct {
    // Name of the cable driver implementation.
    Name string `json:"name"`

    // Options specifies key/value configuration parameters for the cable driver.
    CableOptions *v1.ConfigMap `json:"options,omitempty"`
}
```

The `ClusterConnectionPolicies` CR defines the criteria for selecting which cluster pairs/subsets can connect and
associated optional configuration options to use for each cluster connection. To resolve the connectivity policy
for a remote cluster's Endpoint, Submariner would:  

- Search for a policy whose left and right label selectors match the corresponding Cluster resources of the local
  and remote Endpoints.

If any matching policy is found, the clusters are either (based on the ConnectionPolicy): connected or not connected.

If more than one matching policy is found, the priority field is used to determine which policy to use. If policies
of equal priority are found it's assumed that these policies are to be applied in parallel.

If no matching policy is found, the default cable driver and default connection policy are used. This policy can be
auto-created at broker deploy time and should specify the cable, the connection policy (to connect or not connect
nodes that don't match predefined policies). If the default policy is not to connect nodes that don't match the
selector criteria, then default cable is not needed.

> **_NOTE:_** In the case of multiple labels being specified, the selector will need to match all the labels.
> Equality-based requirements will be supported. For e.g. labels such as `env: !production` will match any clusters
> that aren't labelled with `env: production`.

Defining separate resources for each policy and defining a priority or precedence field, allows Submariner to prioritize
the connectivity and cable type between a pair of clusters.

The initial `ClusterConnectionPolicies` resources would be created (at startup time) and maintained on the broker cluster
and synced to each participating cluster. Policies can subsequently be created/deleted on a cluster and exported to the broker.
A controller to reconcile updates to policies will need to be implemented in Submariner.

### Labelling clusters

Clusters are defined as Custom Resources and can be labeled in the CRD. A new parameter to label clusters at
`subctl join` time could also be added for usability purposes.

### Policy management with subctl

The `ClusterConnectionPolicies` can be created at the Broker deployment through the extension of Submariner configuration
YAML to include parameters that support policy creation.

The `ClusterConnectionPolicies` can be created/managed after Broker deployment through subctl using a command such as:

```cmd
subctl connection-policy add | list | delete
```

> **_NOTE:_** Policy CRs are automatically synced to the Broker on creation.

#### subctl connection-policy commands

##### `add`

`subctl connection-policy add [flags]`

<!-- markdownlint-disable line-length -->
| Flag                                  | Description
|:--------------------------------------|:--------------------------------------------------------------------------|
| `--kubeconfig` `<string>`             | Absolute path(s) to the kubeconfig file(s) (default `$HOME/.kube/config`)
| `--kubecontext` `<string>`            | Kubeconfig context to use
| `--name` `<string>`                   | The name of this policy (used in the metadata)
| `--left-cluster-selector` `<string>`  | Comma separated list of cluster labels for the label selector to use
| `--right-cluster-selector` `<string>` | Comma separated list of cluster labels for the label selector to use
| `--priority` `<value>`                | The priority of this policy
| `--connection-policy` `<string>`      | The connection policy: connect/no-connect
| `--cable-driver` `<string>`           | The name of the cable driver to use for this policy
| `--cable-config`  `<string>`          | The cable driver ConfigMap name
<!-- markdownlint-enable line-length -->

##### `delete`

`subctl connection-policy delete [flags]`

| Flag                       | Description
|:---------------------------|:-------------------------------------------------------------------------------|
| `--kubeconfig` `<string>`  | Absolute path(s) to the kubeconfig file(s) (default `$HOME/.kube/config`)
| `--kubecontext` `<string>` | Kubeconfig context to use
| `--name` `<string>`        | The name of the policy to delete

##### `list`

`subctl connection-policy list [flags]`

Inspects the cluster and reports information about the detected connection policies.

| Flag                         | Description
|:-----------------------------|:----------------------------------------------------------------------------|
| `--kubeconfig` `<string>`    | Absolute path(s) to the kubeconfig file(s) (default `$HOME/.kube/config`)
| `--kubecontext` `<string>`   | Kubeconfig context to use

#### Connection Policy Examples

- we want to specify a default connection policy that uses "full-mesh" with IPsec as the default cable driver.

```yaml
kind: ClusterConnectionPolicies
metadata:
  name: public-internet
spec:
  connectionPolicy: connect
  cableDriver:
    name: "ipsec"
  priority: 0
```

- we want to specify a default connection policy that uses a doesn't connect clusters.

```yaml
kind: ClusterConnectionPolicies
metadata:
  name: public-internet
spec:
  connectionPolicy: no-connect
  priority: 0
```

- We want to use specific cable drivers to connect clusters in a ClusterSet as shown in the image below. We
label the `Clusters` appropriately and define a number of policies with the appropriate label selectors.

![Finer grained connectivity example](./images/multiple_cables_ep.png)

```yaml
kind: ClusterConnectionPolicies
metadata:
  name: on-prem
spec:
  leftClusterSelector:
    location: on-premise
  rightClusterSelector:
    location: on-premise
  connectionPolicy: connect
  cableDriver:
    name:  "vxlan"
  priority: 3
```

```yaml
kind: ClusterConnectionPolicies
metadata:
  name: on-prem-to-cloud
spec:
  leftClusterSelector:
    env: production
  rightClusterSelector:
    env: production
  connectionPolicy: connect
  cableDriver:
    name:  "ipsec"
  priority: 3
```

```yaml
kind: ClusterConnectionPolicies
metadata:
  name: cloud-a-default
spec:
  leftClusterSelector:
    location: cloud
  rightClusterSelector:
    location: cloud
  cableDriver:
    name:  "vxlan"
  priority: 3
```

```yaml
kind: ClusterConnectionPolicies
metadata:
  name: default-policy
spec:
  connectionPolicy: no-connect
  priority: 0
```

- We don't want to connect two clusters in a `ClusterSet`. We label the `Clusters` appropriately and define
  a policy with the appropriate label selectors.

```yaml
kind: ClusterConnectionPolicies
metadata:
  name: on-prem-to-cloud-no-connect
spec:
  leftClusterSelector:
    location: on-premise
    env: production
  rightClusterSelector:
    location: cloud
    env: production
  connectionPolicy: no-connect
  priority: 3
```

- We only want certain clusters to access a database service in another cluster and no client clusters to connect
  to one another. We label the `server` and `client` `Clusters` appropriately and define the following policies:

```yaml
apiVersion: v1
kind: ClusterConnectionPolicies
metadata:
  name: client-server-policy
spec:
  leftClusterSelector:
    database-role: server
  rightClusterSelector:
    database-role: client
  connectionPolicy: connect
  cableDriver:
      name:  "vxlan"
   priority: 3
```

```yaml
apiVersion: v1
kind: ClusterConnectionPolicies
metadata:
  name: client-policy
spec:
  leftClusterSelector:
    database-role: client
  rightClusterSelector:
    database-role: client
  connectionPolicy: no-connect
   priority: 3
```

- We want any cluster to access a database service in a "server" cluster but no other clusters to connect. We label
  the "server" `Cluster` appropriately and define the following policies, one with the corresponding label selectors and an
  empty selector to match all and a default policy to not connect clusters.

```yaml
apiVersion: v1
kind: ClusterConnectionPolicies
metadata:
  name: client-server-policy
spec:
  leftClusterSelector:
    database-server: true
  rightClusterSelector:
  connectionPolicy: connect
  cableDriver:
      name:  "vxlan"
   priority: 3
```

```yaml
apiVersion: v1
kind: ClusterConnectionPolicies
metadata:
  name: client-policy
spec:
  connectionPolicy: no-connect
   priority: 3
```

### Multiple simultaneous Cable driver configuration to support finer grained connectivity

The cable driver configuration in Submariner only allows for the configuration of a single cable driver for a
gateway. This will pose a challenge when allowing for varying (or multiple) cable types between clusters based
on the defined connectivity policies. The goals of this proposal for the extension of the cable driver configuration
are to:

- Maintain backward compatibility with the current cable driver configuration
- Create an extended cable driver configuration that allows for the setup of one, many or combined drivers.

The proposed approach for additional Submariner cable driver configuration parameters is:

- To continue to use the existing environment variables injected into the Gateway Pod (with the exception of PSKs/secrets).
  These environment variables will come from the `Submariner.io_Submariners.yaml`.
- To extend `Submariner.io_Submariners.yaml` with some new global parameters for multiple cable driver configurations
  in a ClusterSet.
- To use a volume mounted ConfigMap for more detailed cable driver configurations (as these may vary).
- To use mounted secrets for parameters that need to be protected, e.g. the Pre-Shared Key (PSK) and to abandon the use of
  environment variables for secrets.

It's important to consider the benefit of volume mounted ConfigMaps and Secrets versus their use as environment variables
injected into the Pod. With volume mounted maps/secrets, updates are automatically reflected in the Pod, so cable driver
configurations can be updated and acted on without having to restart the Gateway Pod. The cable driver itself may need to
be restarted which may have implications on existing/active traffic flows. The ConfigMap can be marked as `optional` to
avoid blocking Pod startup. Environment variables are not updated after a ConfigMap or Secret update; The whole Pod will
need to be restarted to deal with any configuration changes to environment variables.

> **_NOTE:_** As secrets are used in Submariner it's also critical to ensure the Secrets are encrypted in ETCD.

#### Multiple cable driver configuration

The proposed global configuration parameter extensions to `Submariner.io_Submariners.yaml` to configure multiple cable drivers
are listed in the following table:

<!-- markdownlint-disable line-length -->
| Parameter name              | Description
|:----------------------------|:----------------------------------------------------------------------------|
| cableDrivers                | An array of strings listing the drivers to configure as separate cables.
| combinedCableDrivers        | An array of strings listing the drivers to configure as a combined cable e.g vxlan + IPSEC in transport mode
| cableDriverCustomConfig     | The name of the ConfigMap for custom driver configuration.
| cableDriverCustomConfigPath | The path to the mounted ConfigMap volume.
<!-- markdownlint-enable line-length -->

> **_NOTE:_** the existing `cableDriver` parameter when used in combination with the new parameters can be interpreted as the
default/fallback driver.

The specification additions to `Submariner.io_Submariners.yaml` are shown below:

```yaml
cableDrivers:
  items:
    type: string
  type: array
combinedCableDrivers:
  items:
    type: string
  type: array
cableDriverCustomConfig:
  properties:
    configMapName:
      type: string
    namespace:
      type: string
  type: object
cableDriverCustomConfigPath:
  type: string
```

An example of a ConfigMap being used for additional VXLAN driver configuration is shown below:

```yaml
​​apiVersion: v1
kind: ConfigMap
metadata:
  name: cable-driver-config
  namespace: Submariner-operator
data:
  CableCustomConfig: |
  vxlanID: 100
```
