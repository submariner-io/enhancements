# **Submariner OVN CNI Enhancement for IPv6 Support**

## **Summary**

This proposal outlines the changes required in Submariner for  OVN Kubernetes
CNI to enable IPv6 support, ensuring seamless connectivity between clusters
using Submariner. The main proposal has the full design
[IPv6 Datapath Enhancements](IPV6-datapath.md)
This covers only the OVN CNI part of it.

---

## Design Details

The OVNKubernetes driver programs network policies and routes to direct traffic from
the gateway and non-gateway nodes to the remote cluster.
At present the routes are only programmed for IPv4 addresses. We need to enhance
this to support IPV6 addresses as well.

The handler for creating the Gateway and NonGateway routes needs to be enhanced.

### GatewayRoute CRD

A GatewayRoute CR will be created for each address family supported by the local cluster.
In the case of a dual-stack environment a CR will be created for both IPv4 and Ipv6 addresses.
For IPv6, only an IPv6 GatewayRoute will be created.

The next hop will be the interface IP of ovn-k8s-mp0 interface, which is expected
to have both IPv4 and IPv6 IPs in the case of dual-stack environments.

```yaml
apiVersion: submariner.io/v1alpha1
kind: GatewayRoute
metadata:
  name: remote-cluster-route
spec:
  nextHops:
    - "fd00:abcd::1"
  remoteCIDRs:
    - "fd00:4321::/64"
```

### NonGatewayRoute CRD

The NonGatewayRoute will follow the same pattern as GatewayRoute with the creation of a new CR
for IPv6. The nexthops will be the transit switch IP of the gateway node.

#### **NonGatewayRoute CRD Example**

```yaml
apiVersion: submariner.io/v1alpha1
kind: NonGatewayRoute
metadata:
  name: non-gw-route
spec:
  nextHops:
    - "fd00:cafe::1"
  remoteCIDRs:
    - "fd00:5678::/64"
```

### GatewayRoute Handler

The GatewayRoute Handler should be aware of the IPv6 address that can be present in the CR
and program the logical router policy and the logical route accordingly.

The below is the logical router policy to reroute the submariner traffic to ovn-k8s-mp0.

```plaintext
match: "ip6.dst==fd00:5678::/64"
action: reroute
nexthops: ["fd00:abcd::1"]
priority: 20000
```

The below is the logical route to accept the traffic coming from non-gateway nodes.

```plaintext
destination: "fd00:1234::/64"
nexthop: "fd00:cafe::1"
priority: 200
```

### NonGatewayRoute Handler

The NonGatewayRoute Handler should be aware of the IPv6 address that can be present in the CR
and program the logical router policy accordingly.

The below is the logical router policy to reroute the submariner traffic to transit switch
connecting to the gateway node.

```plaintext
match: "ip6.dst==fd00:5678::/64"
action: reroute
nexthops: ["fd00:abcd::1"]
priority: 20000
```

### TODO

* Enhance GatewayRoute controller and NonGatewayRoute controller to support IPV6
* Ensure that GatewayRoute Handler and NonGatewayRoute Handler are programming the
required routes, if not make the required changes.

---
