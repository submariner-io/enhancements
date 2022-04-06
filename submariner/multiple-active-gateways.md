# Multiple Active Gateways

## Summary

NOTE: The design details are still a work in progress, so this is currently a draft proposal.

Submariner currently only allows a single gateway to be active at any one time in a given cluster.
This enhancement proposes that there be an option to allow more than one gateway to be active in
a given cluster to enhance both performance and resiliency. This feature is sometimes referred to as
“active-active gateways”; however, the goal is to allow two or more active gateways to be used.

## Proposal

The proposal is to optionally allow multiple gateways to be active at the same time in a given cluster.

When only a single active gateway is used, all inter-cluster traffic must flow through that one gateway,
which creates a performance bottleneck. Enabling more than one node to be used as a gateway can improve
the performance linearly in the number of gateways used.

Additionally, when the active gateway fails in the current implementation, work must be done to switch
over to the new gateway including the establishment of new tunnels (both intra-cluster and inter-cluster)
and new routes, which takes time. With the proposed design, It is expected that the fail-over times can
be improved if additional gateways are operational. Furthermore, it is expected that
traffic flowing through operational gateways will continue to flow while the traffic from the failed
gateway is redirected to the operational gateways. Both properties reduce the impact of a failure.

## Design Details

### Single Gateway

In the current single gateway Submariner implementation, intra-cluster VXLAN tunnels are established by
the non-gateway nodes to the gateway node, and each gateway establishes an inter-cluster tunnel to each
other gateway in the clusterset (with the tunnel type determined by the cable driver in use) as shown
below.

![SingleGateway](./images/single-gateway.png)

### Multiple Active Gateway Design

The proposal for supporting multiple active gateway nodes is shown in the following diagram and described below.

![MultiGateway](./images/multi-gateway.png)

#### Summary of Data Plane Changes
* Each non-gateway node will establish a VXLAN tunnel to each gateway node.
* The non-gateway nodes will load-balance flows between the active gateway nodes.
* Each gateway node will establish a tunnel to each peer gateway node.
* The gateway nodes will load-balance flows between the active peer gateway nodes.

#### Load balancing options
Multiple options for load balancing exist.
* Option 1: The one currently supported by vishvananda/netlink, and implemented in the POC, is to include multiple next hops in
each route using the "nexthop" option.
* Option 2: A more efficient option is to use nexthop groups.  They are more efficient because multiple routes can use the same
nexthop group, and when the nexthop group changes, you only need to update the group object and not all of the routes
that use it. A good discussion can be found
[here](https://lpc.events/event/4/contributions/434/attachments/251/436/nexthop-objects-talk.pdf).
* Option 3: Resilient nexthop groups are an even better option because they reduce the churn that happens when flows hash to new
links after a nexthop group changes. A good overview of resilient nexthop groups can be found
[here](https://docs.kernel.org/networking/nexthop-group-resilient.html).

nexthop groups are not currently supported by vishvananda/netlink, so that support would need to be added upstream
and then pulled into Submariner before it makes sense to use them for Submariner.  Furthermore, resilient next hop group
support was only added in kernel version 5.13 and iproute2 version 5.10, so a fairly recent Linux distribution is required.
Therefore, we would most likely use Option 1 in the short term.

### Multiple Active Gateway Packet Flow

An example packet flow is shown in the following diagram and described below.

![MultiGateway](./images/multi-gateway-flow.png)

1. Node 1-1
    1. Intercept packet destined for global service
    2. Load balances flow to gateway 1-1
    3. Hash ensures remaining packets in the connection also go to gateway 1-1
2. Gateway 1-1
    1. Determines packet is destined for Cluster 2 based on dest Globalnet IP
    2. Load balances flow to Gateway 2-2
    3. Perform SNAT from local source/port to one of the Globalnet address/ports for gateway 1-1
    4. Hash ensures that remaining packets in the flow also go to Gateway 2-2
3. Gateway 2-2
    1. The packet is handed off to the local service load balancer, which performs necessary NAT
operations and forwards the packet to chosen endpoint on Node 2-1.
    2. Ingress pPackets are SNAT'ed when transmitted on the CNI interface so that they are directed back to
the ingress gateway node. An option (currently implemented in the POC) is to use the CNI interface address.
However, we'll probably need a solution with more than one IP address to support more than 64K flows.
This is required because the state used to SNAT the reply packets back to the global service IP only exists
on the ingress node.
4. Node 2-1
    1. Send response back to Gateway 2-2 (through service load balancer)
    2. Note: it should be okay if the return path uses a different gateway, but I think the cluster
load balancer will return it to the same node in order to maintain the NAT state.
5. Gateway 2-2
    1. Forward back to Gateway 1-1 over the appropriate tunnel
    2. Note: It is important that Gateway 2-2 forwards the packet directly to Gateway 1-1 and not
via the nexthop load balancer which may send it to Gateway 1-2. This can be accomplished by using host
routes for the gateway IP addresses that are higher priority than the routes for the cluster Globalnet
CIDRs which would use the nexthop load balancing.
6. Gateway 1-1
    1. Perform DNAT from the global address back to the local address
    2. Send to the pod on Node 1-1 via local CNI.

### Control Plan Changes

Significant changes are required to the Globalnet control plane to remove assumptions that there is a single gateway.

Details will follow in a subsequent update.

## Scaling considerations
* Additional tunnels between clusters are required for this design.

## Notes and Assumptions

* This design assumes that Globalnet is being used. It may be possible to make it work without Globalnet if
necessary, but step 5 relies on the fact that the dest IP will be the address for the correct gateway.
Without Globalnet, we’d still need to do the same type of SNAT operation at step 2 using local addresses
which isn’t currently done by Submariner.

* The initial POC will use standard external IP addresses for gateways, but load balancer mode is
a high priority for the final implementation.

* The initial design and POC is focused on CNIs that use the kubeproxy route agent handler that establishes VXLAN
tunnels between worker nodes and gateways. Other options need to be investigated and supported.

* There will be a config option to enable multiple active gateways, and when enabled, all nodes labeled with
“submariner.io/gateway=true” will become active. It is for future study whether we should support an option
where we support more than one active gateway as well as additional backup inactive gateways.

* The multipath routes require an interface, so this approach won't work with IPsec Tunnel Mode.  IPsec Transport mode,
proposed in [pr98](https://github.com/submariner-io/enhancements/pull/98) will be required for IPsec encryption.

## TODO
* Support non-kubeproxy route agent handlers.
* Support Loadbalancer mode.
* Support detection and handling of gateway failure and recovery.
* Add more details on control plane design.
* Add more details on the data plane design.

## Plan

The plan is to implement a POC based on this design, validate the benefit, and then update the design and proposal
as needed.
