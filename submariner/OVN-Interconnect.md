# Submariner Enhancement for OVN Interconnect

<!-- Add link to issue/epic if available -->

## Summary

OVN CNI is moving to a new network topology powered by OVN Interconnect Feature. This allows independent
OVN deployments to be interconnected by OVN-managed GENEVE tunnels. Submariner does not work with these changes
out of the box and this proposal explains the changes required to support it.

## Proposal

With OVN Interconnect we can have two types of deployment

### Single Zone(Global Zone)

A single-zone deployment will have only one OVN database and a set of master nodes programming it. This is similar to the topology that we
use now. But it will have a Zone(global) assigned to it and will have a transit switch created for it. Though the transit switch is not
involved in any packet forwarding in this topology. Though this setup is expected to work by default, the subnet that the transit switch
used overlaps with the subnet Submariner uses for the logical router. So we need to agree upon a non-overlapping subnet for Submariner.

### Multiple Zone

In a multiple-zone setup, we will have an OVN database and a set of master nodes for each zone. Transit switches connect between these
nodes. The OVN-Kubernetes services ensure that the necessary routes are added for pod and service reachability across nodes in different
zones.

With the current architecture, Submariner adds routes only in the zone in which it is deployed. For example, if Submariner is deployed in
zone 1 it programs OVN db in zone 1. So only pods in zone 1 nodes will be able to talk to other clusters. Pods in zone 2 or zone 3 will not
be able to reach remote clusters connected via Submariner.

In this proposal, we explore two options to make Submariner work with OVN IC. More details about the upgrade and other design details shall
be added once agree on one of them.

### Option1: Add extra routes in existing Architecture

 This proposal makes minimal changes in the network plugin syncer to get OVN IC working. With this, we add new routes in the OVN cluster
router.

![UpdatedTopology](./images/submariner-ovn-ic.png)

These are the changes that will be needed

1) Network plugin Syncer needs to be Zone aware
2) In the Zone where Submariner Gateway is present an extra route will be added to direct the traffic to remote CIDRS connected via
submariner to the Submariner router. This is for the remote traffic coming from other zones.
3) In all the other zones a route will be added to direct the traffic to remote cluster CIDRS, connected via submariner, to the transit
switch that connects to the gateway node

#### Node Listener

A new node listener shall be added in NetworkPluginSyncer.

* The node listener shall be responsible for reading the node labels/annotations and identifying the zone in which the Submariner gateway is
deployed and the transit switch IP for it.
* When a new node is added or a node joins a zone, the node listener needs to call the NetworkPluginSyncer handler to program the required
rules. It will also remove any existing stale rules.
* When a node leaves no action is needed
* When the gateway nodes join a new zone, leaving the current one, all the switches will need to be reprogrammed as the transit switch IP
can change.

#### Network Plugin Syncer

Based on the info retrieved by the Node listener, the network plugin syncer can provision the Submariner infrastructure and program the
rules in the OVN database.

* If no zone is detected(will be required for backward compatibility) or a global zone is detected the Submariner infrastructure and flows
will remain as it is today.

* If a zone configuration is detected

1) In the gateway node zone a new routing rule can be added to forward the traffic destined for Submariner remote CIDRS
to the Submariner router. This is required for the traffic coming from pods in a node that is in a different zone.

2) In the non-gateway zones a route will be added for the traffic destined for remote cluster CIDR to forward it to the gateway node
transit switch.

#### Pros of Option1

This makes the least changes than option 2 and can be delivered within the time frame we have.

#### Cons of Option1

It depends on OVN labels/annotations for programming the routes, which means a change in these would break Submariner. Also given the plans
to change the OVN -Submariner architecture this has to be reworked in future.

### Option2: Submariner OVN Re-architecture

There is an effort to optimize Submariner OVN integration. As a part of this Submariner Router and related ports and join switch shall be
removed.  The OVN gateway router needs to be configured not to SNAT the traffic destined to local Cluter CIDR/ Global net CIDRS(This is now
the default behaviour of OVN, we need it to test it though).

Further, with this optimization Submariner should stop configuring the OVN cluster router with Logical Route policies using OVN APIs
directly. OVN should expose CRD, which Submariner can use to program the OVN cluster router with the required Logical router policy and
routes. If done, the Submariner need not be aware of the OVN infrastructure details like zones and transit switches. An OVN plugin (or a
component in OVN) can consume the CRD and program the routing rules and provision the infrastructure required by Submariner.

While this is a cleaner and maintainable implementation it will not be possible to deliver it by the next Submariner release. Also, we need
to agree upon the APIs required in OVN and their implementation.

#### Pros of Option2

This will lead to a more maintainable code and fewer moving parts and this is how we plan Submariner-OVN integration will be in future.

#### Cons of Option2

It takes more time to implement this option. Submariner changes need to wait until OVN changes required for Submariner are done.

## Design Details

### Backward Compatibility

<!-- Any backward compatibility concerns -->

### Alternatives

#### Route Advertisement

Another alternative was explored which could leverage the [route advertisement](
https://docs.ovn.org/en/latest/tutorials/ovn-interconnection.html#route-advertisement) feature in OVN. The idea was to use this capability
to advertise Submariner routes. But this feature seems to be not available in the OVN Kubernetes IC implementation.

## External Dependencies

<!-- Any external dependencies this proposal may have -->

## User Impact

<!-- Optional. Any impact on users this change may have. -->
