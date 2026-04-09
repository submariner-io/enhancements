# Submariner Enhancement for OVN Interconnect

<!-- Add link to issue/epic if available -->

## Summary

OVN CNI is moving to a new network topology powered by OVN Interconnect Feature. This allows independent OVN deployments,
within a kubernetes cluster, to be interconnected by OVN-managed GENEVE tunnels. Submariner does not work with these changes
out of the box and this proposal explains the changes required to support it.

## Proposal

With OVN Interconnect we can have two types of deployment

* Single Zone(Global Zone): A single-zone deployment will have only one OVN database and a set of master nodes programming it. This is similar
  to the topology that we use now. But it will have a single `global` zone assigned to it with a transit switch which is not involved in any
  packet forwarding. Though this setup is expected to work by default, the subnet that the transit switch uses overlaps with the subnet
  Submariner uses for the logical router. In this proposal, we plan to remove the submariner logical router and the associated switches and
  this will not be a problem anymore.

  The below indicates node annotations in a single zone OVN setup and is added by OVN.

```bash
    annotations:
      k8s.ovn.org/ovn-node-transit-switch-port-ips: '["169.254.0.1/16"]'
      k8s.ovn.org/ovn-zone: global
    name: cluster1-worker

    annotations:
      k8s.ovn.org/ovn-node-transit-switch-port-ips: '["169.254.0.1/16"]'
      k8s.ovn.org/ovn-zone: global
    name: cluster2-worker
```

* Multiple Zone: In a multiple-zone setup, we will have an OVN database and an OVN master pod for each zone. Transit switches connect the
  zones. The OVN-Kubernetes services ensure that the necessary routes are added for pod and service reachability across nodes in different
  zones.

```bash
    annotations:
      k8s.ovn.org/node-transit-switch-port-ifaddr: '["169.254.0.3/16"]'
      k8s.ovn.org/zone-name: global
    name: cluster1-worker

    annotations:
      k8s.ovn.org/node-transit-switch-port-ifaddr: '["169.254.0.5/16"]'
      k8s.ovn.org/zone-name: az2
    name: cluster2-worker
```

With the current architecture, Submariner network-plugin-syncer adds routes only in a single zone where the network-plugin-syncer pod runs.
For example, if network-plugin-syncer is deployed in zone 1 it programs OVN db in zone 1. So only pods in zone 1 nodes will be able to talk
to other clusters. Pods in zone 2 or zone 3 will not be able to reach remote clusters connected via Submariner.

As part of the proposal, we plan to support both the modes and OVN cluster deployments where interconnect
is not enabled as well.

### Route APIs

As part of this proposal, we are planning to add two new CRDs in Submariner. The plan is to decouple OVN dataplane programming so that this
can be moved to OVN, when OVN provides a way to add custom routes. We can propose this CRD or consume the CRD that they provide at a later
point of time.

#### GatewayRoute

This CR will be created when a remote endpoint is added and there will be one CR per remote cluster. This CRD has two fields

* NextHops - Specifies the list of next hop to reach the remote cluster, in this case it will be the IP of ovn-k8s-mp0
  interface, the interface used by OVN for host networking.
* RemoteCIDR - Specifies the list of  remote CIDRs reachable via the next hop.

This CR will be used by the route agent pod running on the active-Gateway node to program OVN to send the traffic destined to
remote clusters via the Submariner tunnel. Also it adds a route to send the traffic from other zones, destined to remote cluster
to the submariner tunnel

``` go
type GatewayRoute struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    RoutePolicySpec RoutePolicySpec `json:"spec"`
}

type RoutePolicySpec struct {
    // Specifies the next hops to reach the remote CIDRs
    NextHops []string `json:"nextHops"`

    // Specifies the remote CIDRs available via the next hop
    RemoteCIDRs []string `json:"remoteCIDRs"`
}
```

#### NonGatewayRoute

This CR will be created when a remote endpoint is created and there will be one created per endpoint.

* NextHops - Specifies the list of next hops. In this case ,we will have only one, and it will be the transit switch IP of the zone
where g/w node is present.
* RemoteCIDR - Specifies the list of remote CIDRs reachable via this gateway.

* In non-g/w node - If the route-agent pod is not in the same zone as Gateway node zone, send the traffic to the g/w node zone.

``` go
type NonGatewayRoute struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    RoutePolicySpec RoutePolicySpec `json:"spec"`
}

type RoutePolicySpec struct {
    // Specifies the next hops to reach the remote CIDRs
    NextHops []string `json:"nextHops"`

    // Specifies the remote CIDRs available via the next hop
    RemoteCIDRs []string `json:"remoteCIDRs"`
}
```

## Design Details

In this proposal, the plan is to remove the current `submariner-router` and switches that were added to the OVN db. The network-plugin-syncer
shall be removed  and can be replaced by controllers in and Submariner RouteAgent. Since we have multiple ovn db to
program, we need multiple connections. So it makes it easier to program the OVN datapath from RouteAgent, than using
a separate pod. This new approach will work for both IC enabled and existing deployments. With this change we are planning to use the
ovn-k8s-mp0 interface to reach the host networking stack and then the Submariner tunnel. This interface is used by OVN for host-networking
traffic with in a cluster and will be present in every node.

![UpdatedTopology](./images/submariner-ovn-ic.png)

### SubmarinerRouteAgentPod

The Submariner Route-agent pod running on the active gateway node will be responsible for creating the GatewayRoute CRs. It will be used
only for OVN CNI. For every RemoteEndpointCreated event a GatewayRoute CR will be created. The nextHop will be the interface IP through
which we can reach the cable driver. In the case of OVN it will be the IP of ovn-k8s-mp0 interface.

The NonGatewayRoute CRD will also be created by Submariner Route-agent running on the active gateway node . It will be created per endpoint
and will have remoterCIDRS from the endpoint. The nextHop will be the transit switch IP of the G/W node. If the transit switch IP is missing
this CR will not be created, which means it is a non-IC setup.

The RouteAgent  will have these controllers added to it and the one running in gateway node responds to the CRUD operations of Submariner
endpoints.

#### GatewayRoute CR Controller

This controller will be responsible for programming the OVN cluster router, it will react only the active g/w node. When a GatewayRoute
CR is created or modified the controller shall create or update a routing policy in OVN cluster router with a priority of 20000, and it should
redirect any traffic destined to remote CIDR to the ovn-k8s-mp0 interface IP.

```bash
_uuid               : 0459f009-3603-47ac-8ee7-9d958540ed31
action              : reroute
external_ids        : {}
match               : "ip4.dst==10.132.0.0/16"
nexthops            : ["10.1.1.2"]
options             : {"external_ids:{submariner"="true}"}
priority            : 20000
```

It also programs a route in the ovn-cluster-router, to route the traffic coming from other zones destined to remote cluster IP range via the
ovn-k8s-mp0 interface IP.

```bash
_uuid               : d55185d8-3732-45c1-ae90-4a7f8cd191f7
bfd                 : []
external_ids        : {}
ip_prefix           : "10.132.0.0/16"
nexthop             : "10.1.1.2"
options             : {}
output_port         : []
policy              : []
route_table         : ""
```

#### NonGatewayRoute Controller

This controller will run as part of every route agent pod and connects to the OVN DB. When a NonGatewayRoute CR is created,
the route-agent running on the non-GW node will update the ovn-cluster-router with a logical router policy using a priority of 20000
to send the traffic to the remote cluster via next hop mentioned, which is the transit switch IP to the g/w node. Before adding
the route it checks if a route exists, if so it skips adding the route again. This is required to prevent duplicate update since there
can be more than one node in each zone and hence more than one RouteAgent.

```bash
_uuid               : 22db3005-64c5-4e32-aeb0-642423c30742
action              : reroute
external_ids        : {}
match               : "ip4.dst==10.132.0.0/14"
nexthop             : []
nexthops            : ["169.254.0.1"]
options             : {"external_ids:{submariner"="true}"}
priority            : 20000
```

### Backward Compatibility

With this architecture, we are removing the logical switches and routers that Submariner created in the existing implementation.
During migration, we will need to delete these components and the routes that network-plugin-syncer installed. After that,
the new controllers shall install the updated routes. The operator needs to be updated as well, not to create a
network-plugin-syncer pods and remove any existing deployments.

### Gateway FailOver

If there are two gateway nodes , the passive one will work like a non-gateway node. It will not be responsible for creating the CRs.
In the case of gateway fail-over all the current GatewayRoute and NonGatewayRoute will be deleted by the route agent in
the node that is transitioning to gateway node and will be recreated with updated values.

#### Open issues

1. When multiple clusters are updated we need to check if one cluster can be done at a time. The cluster will be down
    until all the nodes are updated.
2. The update from older version of Submariner to a newer version will create  a datapath downtime.
3. When Kubernetes cluster is updated to a version that has IC enabled, there could be a datapath downtime till the
   Submariner g/w node is updated. Since the other nodes need the transit switch IP which will be available only when
   the g/w node is updated.
4. Explore the possibility  of VIP to represent the gateway node switch IP instead of reconfiguring all the routes at non-gw nodes

### Alternatives

#### Adding extra routes

We explored a proposal for adding an extra routes in the existing architecture. This would mean adding same extra routes as in this proposal
but by maintaining the submariner switches and router. While this should be easier to implement this be maintenance overhead in the future.

#### Route Advertisement

Another alternative was explored which could leverage the
[route advertisement](https://docs.ovn.org/en/latest/tutorials/ovn-interconnection.html#route-advertisement)
feature in OVN. The idea was to use this capability to advertise Submariner routes. But this feature seems to be not available in the
OVN Kubernetes IC implementation.

## External Dependencies

These OVN changes should be merged for OVN Interconnect support in the CNI

[Add cluster manager support](https://github.com/ovn-org/ovn-kubernetes/pull/3127)

[Add zone support](https://github.com/ovn-org/ovn-kubernetes/pull/3169)

[Multiple zone support](https://github.com/ovn-org/ovn-kubernetes/pull/3366)

## User Impact

The user should be able to use Submariner in OVN enabled setups as well the non-IC ones. The pods that the user see when a Submariner is deployed
will change and similar change can be expected in diagnose command outputs.
