# Node Placement

[Add support for nodeSelector and toleration scheduling for submariner pods](https://github.com/submariner-io/enhancements/issues/149)

## Summary

Submariner deploys many different agents as part of its core deployment. Gateway, Globalnet Agent and
Metrics Proxy are deployed on designated gateway nodes, while Route Agents are deployed on all nodes. There is no way
for users to control or limit where the rest of the agent pods are deployed. Some users want to run these agents on
specific nodes for billing or networking reasons. Users can't add tolerations directly to these agents because they'll be
overwritten by the operator. So, users require an easily configurable way to specify these parameters.

## Proposal

This enhancement proposes adding a new field to `Submariner` and `ServiceDiscovery` for users to configure node
scheduling. This configuration will then be applied to pods of the following submariner agents:

* submariner-operator
* submariner-lighthouse-agent
* submariner-lighthouse-coredns
* submariner-networkplugin-syncer

Following agents will be excluded because these are run on Gateway nodes. Users already control which nodes get labeled
as gateways and ideally we don't want to run anything else on these nodes.

* submariner-gateway
* submariner-globalnet
* submariner-metrics-proxy

## Design Details

Add new `NodeSelector` and `Tolerations` fields to `Submariner` and `ServiceDiscovery` specs.

```Go
type SubmarinerSpec struct {

    // NodeSelector defines which Nodes the Pods are scheduled on. The default is an empty list.
    // +optional
    NodeSelector map[string]string `json:"nodeSelector,omitempty"`

    // Tolerations is attached by pods to tolerate any taint that matches
    // the triple <key,value,effect> using the matching operator <operator>.
    // The default is an empty list.
    // +optional
    Tolerations []v1.Toleration `json:"tolerations,omitempty"`
}
```

```Go
type SubmarinerDiscoverySpec struct {

    // NodeSelector defines which Nodes the Pods are scheduled on. The default is an empty list.
    // +optional
    NodeSelector map[string]string `json:"nodeSelector,omitempty"`

    // Tolerations is attached by pods to tolerate any taint that matches
    // the triple <key,value,effect> using the matching operator <operator>.
    // The default is an empty list.
    // +optional
    Tolerations []v1.Toleration `json:"tolerations,omitempty"`
}
```

Users can add these fields to CR manually through `kubectl`. Operator code will copy over these values to spec for
applicable Pods.

### Pros

* Provides easy way for users to configure node scheduling

### Cons

* None

### Backward Compatibility

None

### Alternatives

* One alternative is to provide means to add tolerations and node selectors to agent pods directly without being
those being overwritten by operator. This was rejected because it breaks the operator design philosophy.

## External Dependencies

None

## User Impact

None
