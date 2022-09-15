# Subctl `diagnose pre-install`

## Summary

As a user of Submariner, it would be useful in a myriad of situations to have a tool capable of testing the Kubernetes infrastructure
where Submariner is to be installed.
The tool should check all of Submariner's pre-requisites from the infrastructure, and report any deficiencies.
It would be nice if the tool can also offer guidance for the deficiencies, so that the user can address them and install Submariner successfully.

This tool would compliment other `subctl diagnose` commands which check that Submariner is installed and working properly, by allowing
similar checks **before** the actual installation.

Alternatively, the new command could be hosted elsewhere, if nesting under `diagnose` is deemed confusing.

## Proposal

Add a `subctl diagnose pre-install` command which checks that the infrastructure supports installing Submariner by checking:

* Gateway nodes are set-up.
  * Kube proxy, if there, doesn't run in IPVS mode - as it's currently not supported.
* The broker is reachable from the cluster.
* K8s version is supported.
* The CNI is supported.
* Conflicts with existing clusters in the clusterset.
  * CIDRs should be distinct, unless the clusterset is using globalnet.
* The necessary ports are open.
  * On the gateway, when not using Load Balancer, the cable driver port(s).
  * On the nodes, when not using OVN, the vxlan port.

The broker/clusterset checks could be optional and run only if a broker file is supplied.

Alternatively, instead of adding a new command under the `diagnose` commands, these checks could be done as part of`subctl join`.
This would make sense, since at the point of running the command, we need some information from the user:

* Broker info file.
* Cable driver ports.
* Is load balancer being deployed.

Some of these checks can always be done as part of `join`, while others can be done only when using a flag, e.g. `--check-infra`.

## Design Details

We can reuse existing code in `subctl diagnose` commands as is, or refactor as necessary:

* Gateway node (New check)
  * Check that we have at least one node labeled as gateway.
  * This check is fundamental for firewall checking.
    * Unless we're changing deployment model not to use labeled gateway, in which case those checks will have to be updated anyhow.
  * Kube Proxy Mode (Reuse as is)
    * Run "ip a s kube-ipvs0" on a gateway node and check that the interface isn't there.
    * "The cluster is deployed with kube-proxy ipvs mode which Submariner does not support"
* Broker connectivity (New check)
  * Make sure the broker is reachable from the cluster.
* K8s Version (Reuse as is)
  * Query K8s and check we have a minimal supported version (1.17).
* CNI (Reuse if possible with refactor)
  * Check if CNI as detected by us is supported (from a list of plugins).
    * Reuse submariner-operator/pkg/discovery/network -> Discover
  * Warn that Submariner might not work if the CNI not detected (but don't fail the check).
  * Extra checks for:
    * OVN-K:
      * Version of NB DB is supported.
    * Calico:
      * IP Pools used on node labeled as gateway are disabled.
* Overlapping CIDRs (New check)
  * Check that the CIDRs aren't overlapping with any known CIDRs from the broker.
  * If the clusterset is using globalnet, skip this check.
* Firewall ports (Might be possible to reuse, at least part of it)
  * Internal VxLAN port:
    * Skipped on OVN-K
    * Would be possible to do with:
      * Spawn on a gateway labeled node a pod running `tcpdump -ln -c 5 -i any udp and src port 9898 and dst port 4500`
      * Spawn on non-gateway a pod running `for i in $(seq 10); do timeout 2 nc -u -p 9898 <gatewayPodIP> 4500; done`
  * Public tunnel port:
    * Skipped when deploying with Load Balancer
    * Quite problematic to reuse as we don't necessarily have the public IP, we would need to figure out how to deduce it.
    * Also the existing diagnose check requires a second cluster...
      * Although we could try to initiate from the same cluster to an external IP, but the success depends on the routing policy.
    * Spawn on a gateway labeled node on the local cluster a pod running
      `tcpdump -ln -Q in -A -s 100 -i any udp and dst port <dest port> | grep '<client message>'`
      where the client message is a random string (part of UUID).
    * Spawn on a non gateway node on the local cluster running
      `echo <client message> | for i in $(seq 5); do timeout 2 nc -n -p 9898 -u <gateway ip> <dest port>; done`
      where the gateway ip is the public IP of the pod running the "sniffer"
  * NATT port - Same as tunnel port:
    * Skipped when deploying with Load Balancer
    * Same exact check as the tunnel port, but with the NATT port number

### Pros

Users will have a tool to check if the infrastructure can support Submariner before actually trying to install it, saving them time and
headaches trying to debug issues only post-installation.

### Cons

We would have more code to maintain (although sharing with existing diagnose commands could minimize the impact).

### Backward Compatibility

As this is a new command, no impact on existing behavior is expected.

### Alternatives

We could simply focus our efforts on expanding `subctl diagnose` to cover more cases instead, at the cost of forcing users to first install
Submariner and only then trying to find out why it doesn't work as expected.

## External Dependencies

None.

## User Impact

Positive impact to allow users to test their infrastructure before installation.
