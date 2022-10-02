# Expanded Checks for `subctl join`

## Summary

As a user of Submariner, it would be useful in a myriad of situations to have a tool capable of testing the Kubernetes infrastructure
where Submariner is to be installed.
The tool should check all of Submariner's pre-requisites from the infrastructure, and report any deficiencies.
It would be nice if the tool can also offer guidance for the deficiencies, so that the user can address them and install Submariner successfully.

## Proposal

Expand `subctl join` to check additional pre-requisites from the infrastructure itself, and make sure it supports installing Submariner:

* Gateway nodes are set-up.
  * Kube proxy, if there, doesn't run in IPVS mode - as it's currently not supported.
* The broker is reachable from the cluster.
* Conflicts with existing clusters in the clusterset.
  * CIDRs should be distinct, unless the clusterset is using globalnet.
* The necessary ports are open.
  * On the gateway, when not using Load Balancer, the cable driver port(s).
  * On the nodes, when not using OVN, the vxlan port.

Add a `--dry-run` flag to `subctl join` in order to allow running just the validations, without installing Submariner on the cluster.

## Design Details

We can reuse existing code from `subctl diagnose` commands as is, or refactor as necessary:

* Kube Proxy Mode (Reuse from `diagnose`)
  * Run "ip a s kube-ipvs0" on a gateway labeled node and check that the interface isn't there.
  * "The cluster is deployed with kube-proxy ipvs mode which Submariner does not support"
* Extra CNI checks (Reuse if possible with refactor)
  * Check if CNI as detected by us is supported (from a list of plugins).
    * Reuse submariner-operator/pkg/discovery/network -> Discover
  * Warn that Submariner might not work if the CNI not detected (but don't fail the operation).
  * Extra checks for:
    * OVN-K:
      * Version of nothbound database is supported.
    * Calico:
      * IP Pools are configured in the cluster to instruct Calico not to perform NAT on the subnets associated with the Pod and Service
        CIDRs of the remote clusters.
* Overlapping CIDRs (New check)
  * Non-globalnet - Check that the CIDRs aren't overlapping with any known CIDRs from the broker.
  * If the clusterset is using globalnet, the CIDR is already tested.
* Firewall ports (Might be possible to reuse, at least part of it)
  * Internal VxLAN port:
    * Skipped on OVN-K.
    * Would be possible to do by:
      * Spawning, on a gateway labeled node, a pod running `tcpdump -ln -c 5 -i any udp and src port 9898 and dst port 4500`
      * Spawning, on non-gateway, a pod running `for i in $(seq 10); do timeout 2 nc -u -p 9898 <gatewayPodIP> 4500; done`
  * Public tunnel port:
    * Skipped when deploying with Load Balancer.
    * We could determine the public IP by the same logic we have right now for endpoints,
      allowing us to test using internal -> public IP connection.
    * The test could possibly fail due to routing/security policy not being set up to allow such connectivity.
    * Possible test:
      * Spawn on a gateway labeled node on the local cluster a pod running
        `tcpdump -ln -Q in -A -s 100 -i any udp and dst port <dest port> | grep '<client message>'`
        where the client message is a random string (part of UUID).
      * Spawn on a non gateway node on the local cluster running
        `echo <client message> | for i in $(seq 5); do timeout 2 nc -n -p 9898 -u <gateway ip> <dest port>; done`
        where the gateway ip is the public IP of the pod running the "sniffer".
  * NATT port - Same as tunnel port:
    * Skipped when deploying with Load Balancer.
    * Same exact check as the tunnel port, but with the NATT port number.

### Pros

Users will have a way to check if the infrastructure can support Submariner when trying to install it, saving them time and
headaches trying to debug issues only post-installation.
The `--dry-run` flag can allow the user to run just the validations, without actually installing Submariner.

### Cons

None.

### Backward Compatibility

As we're extending `join`'s checks, some previously working `join` operations might fail, if the infrastructure can't support them.

### Alternatives

We could focus our efforts on expanding `subctl diagnose` to cover more cases instead, at the cost of forcing users to first install
Submariner and only then trying to find out why it doesn't work as expected.

This is less desirable, however, as it necessitates installing Submariner first, and as such misses the aim of validating the
infrastructure prior to installing Submariner.

## External Dependencies

None.

## User Impact

Positive impact by allowing users to test their infrastructure before or during installation.
