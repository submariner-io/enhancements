## Add support for OVN with pluggable network-plugin support

[submariner#778](https://github.com/submariner-io/submariner/issues/778)

## Summary

Today Submariner supports working with existing network-plugins which are based
on the kube-proxy with iptables backend. Any network-plugin outside of this
case won't work, OVN-Kubernetes is an example.

This is a proposal to support pluggable network drivers, starting with OVN-Kubernetes.

## Proposal

Create `submariner-network-plugin-sync` which will be conditionally deployed by the
`submariner-operator` for network plugins which require it. In the case of OVN it will be
the microservice responsible for synchronizing state with the OVN NBDB. Another example
of network-plugin which could make use of this service in the future would be Calico for
automatic [handling of the IPPool entries](https://submariner.io/deployment/calico/)

Re-architect Submariner route-agent in a way such that different network-plugin
implementations can be supported. The re-architecture should create a re-usable library
which other microservices including the new `submariner-network-plugin-sync` could
make use of for handling common submariner events.

Those drivers would be notified of different events like:

* `Initialization` event

    Called once during startup, to let the plugin configure any necessary settings on the
    system once the route-agent is ready

* `Stop` event

    Called once during stop to let the network plugin cleanup anything necessary during
    at stop if at all.

* `Uninstall` event

    Called once if the route agent is being removed from a node, being Submariner un-installed,
    or the route-agent driver being switched, a full cleanup will be necessary in this case.

* `Transition to Non-Gateway` event

    During Gateway to Non-Gateway events on the specific node may need to perform specific
    cleanups

* `Transition to Gateway` event

    When a node becomes Gateway some settings may need to be adjusted on the node
    by the driver.

* `Creation of a local Endpoint`

* `Creation of a remote Endpoint`

* `Update of a local Endpoint`

* `Update of a remote Endpoint`

* `Deletion of a local Endpoint`

* `Deletion of a remote Endpoint`

* `Node creation` on the cluster

* `Node removal` on the cluster

* ... any other event necessary

In the same way that we have cable-drivers today, the route-agent would become aware
of the underlying network plugin the current cluster uses by using an specific driver.

### Synchronizer implementation

Some parts of the drivers could need a single worker handling specific synchronization
work with the network plugin.

The drivers declare in a flag if they need this type of instance.

The synchronizer instances receive the same exact events, but instead of configuring
the dataplane they talk to the network plugin.

### Dataplane implementation

#### OVN-Kubernetes integration

##### submariner-network-plugin-sync

[OVN-Kubernetes](https://github.com/ovn-org/ovn-kubernetes) is a network-plugin for K8s
that integrates with [OVN](https://github.com/ovn-org/ovn).

OVN has two databases, OVN NorthDB , and OVN South DB. OVN-Kubernetes creates an
`ovn_cluster_router`, two virtual switches per node, and an additional router `GR_xxx`
per node.

In our solution we create an aditional virtual submariner router in OVN, connected
via a switch to the ovn_cluster_router, and then to the external network.

##### submariner-route-agent

In addition to the OVN NBDB handling, we will need to insert additional routes in every
host via the route-agent, in a similar way to what we do for the kubeproxy-iptables
implementation today. The purpose of those route entries is handling Host Network traffic
for the **hostNetwork** pods to be able to reach remote cluster Services and Pods.

Also, for the active gateways we will need to introduce routing rules to handle
traffic coming from remote clusters, which will be directed to the external leg
of the `submariner_router`, for example:

```bash
    ip rule add from 10.132.0.0/14 to 10.128.0.0/14 table 5
    ip rule add from 10.132.0.0/14 to 172.30.0.0/14 table 5
    ip route add default via $SUBM_EXT_PORT_IP_A table 5
```

##### Diagram

![OVN Diagram](pluggable-network-plugin-and-ovn-1.svg)

<!-- Source for the diagram here: https://app.diagrams.net/#G1Mo5How0VGiTU50tyxrPfrXVLQAOjE9j9 -->

##### Proof of concept code

```bash

KUBEMASTERS=$(kubectl get pods -A | grep ovnkube-master | awk '{ print $2 }')

function detect_ovn_pods {

      if [[ "$NBDB_POD" != "" && "$SBDB_POD" != "" ]]; then
        return
      fi

      echo "Detecting ovnkube-master for sbdb and nbdb"

      for master in $KUBEMASTERS; do
        if [[ "$NBDB_POD" == "" ]]; then
                kubectl exec -ti -n openshift-ovn-kubernetes $master -c northd -- ovn-nbctl --timeout=2 show >/dev/null 2>&1 && NBDB_POD=$master
        fi
        if [[ "$SBDB_POD" == "" ]]; then
                kubectl exec -ti -n openshift-ovn-kubernetes $master -c northd -- ovn-sbctl --timeout=2 show >/dev/null 2>&1 && SBDB_POD=$master
        fi

        if [[ "$NBDB_POD" != "" && "$SBDB_POD" != "" ]]; then
                break
        fi
      done

    cat >ovn-pods <<-EOF

    SBDB_POD=$SBDB_POD
    NBDB_POD=$NBDB_POD

EOF
}

[[ -f ovn-pods ]] && source ovn-pods
detect_ovn_pods

function ovn_nbtest {
        kubectl exec -n openshift-ovn-kubernetes -c northd $NBDB_POD -- true
}

function ovn_sbtest {
        kubectl exec -n openshift-ovn-kubernetes -c northd $SBDB_POD -- true
}


# verify if the pods changed (new deployment?)
ovn_nbtest || NBDB_POD=
ovn_sbtest || SBDB_POD=

echo sbdb: $SBDB_POD nbdb: $NBDB_POD

if [[ "$NBDB_POD" == "" || "$SBDB_POD" == "" ]]; then

        detect_ovn_pods
fi


function ovn_nbctl {
        kubectl exec -n openshift-ovn-kubernetes -c northd $NBDB_POD -- ovn-nbctl $*
}

function ovn_sbctl {
        kubectl exec -n openshift-ovn-kubernetes -c northd $SBDB_POD -- ovn-sbctl $*
}

function ovn_nbsh {
        kubectl exec -n openshift-ovn-kubernetes -c northd $NBDB_POD -ti -- bash
}

function ovn_sbsh {
        kubectl exec -n openshift-ovn-kubernetes -c northd $SBDB_POD -ti -- bash
}


SUBM_NODE=$(oc get nodes -l submariner.io/gateway=true  -o jsonpath="{.items[*].metadata.name}")
PHYSNET=$(ovn_nbctl lsp-get-options br-local_$SUBM_NODE)
SUBM_ESWITCH=ext_$SUBM_NODE
SUBM_JSWITCH=submariner_join

CLUS_JOIN_PORT_IP=169.254.34.2/29
SUBM_JOIN_PORT_IP=169.254.34.1/29
SUBM_JOIN_PORT_IP_A=$(echo $SUBM_JOIN_PORT_IP | sed 's/\/29//g')
CLUS_JOIN_PORT_IP_A=$(echo $CLUS_JOIN_PORT_IP | sed 's/\/29//g')


SUBM_EXT_PORT_IP=$(ovn_nbctl --bare --columns=networks find Logical_Router_Port name=rtoe-GR_$SUBM_NODE | sed 's/\.2\/24/\.7\/24/g' )

SUBM_EXT_PORT_IP_A=$(echo $SUBM_EXT_PORT_IP | sed 's/\/24//g' )


SUBM_JOIN_PORT_MAC=$(echo -n 00:60:2f; dd bs=1 count=3 if=/dev/random 2>/dev/null |hexdump -v -e '/1 ":%02x"')
SUBM_EXT_PORT_MAC=$(echo -n 00:60:2f; dd bs=1 count=3 if=/dev/random 2>/dev/null |hexdump -v -e '/1 ":%02x"')
CLUS_JOIN_PORT_MAC=$(echo -n 00:60:2f; dd bs=1 count=3 if=/dev/random 2>/dev/null |hexdump -v -e '/1 ":%02x"')


echo $SUBM_NODE $PHYSNET $SUBM_JOIN_PORT_IP $SUBM_EXT_PORT_IP $SUBM_JOIN_PORT_MAC $SUBM_EXT_PORT_MAC $SUBM_JOIN_PORT_IP_A


ovn_nbctl ls-add $SUBM_JSWITCH

ovn_nbctl lr-add submariner_router
ovn_nbctl lrp-add submariner_router subm_ext_port $SUBM_EXT_PORT_MAC $SUBM_EXT_PORT_IP
ovn_nbctl lsp-add $SUBM_ESWITCH subm_ext_sw_port
ovn_nbctl lsp-set-options subm_ext_sw_port router-port=subm_ext_port
ovn_nbctl lsp-set-type subm_ext_sw_port router
ovn_nbctl lsp-set-addresses subm_ext_sw_port router

ovn_nbctl lrp-add submariner_router subm_join_port $SUBM_JOIN_PORT_MAC $SUBM_JOIN_PORT_IP
ovn_nbctl lsp-add $SUBM_JSWITCH subm_join_sw_port
ovn_nbctl lsp-set-options subm_join_sw_port router-port=subm_join_port
ovn_nbctl lsp-set-type subm_join_sw_port router
ovn_nbctl lsp-set-addresses subm_join_sw_port router

GW_CHASSIS_NAME=$(ovn_sbctl $OVN_SBDB --bare --columns=name find Chassis hostname=$SUBM_NODE)

echo $GW_CHASSIS_NAME

ovn_nbctl lrp-set-gateway-chassis subm_ext_port $GW_CHASSIS_NAME 10

ovn_nbctl lrp-add ovn_cluster_router ovn_cluster_submariner_port $CLUS_JOIN_PORT_MAC $CLUS_JOIN_PORT_IP
ovn_nbctl lsp-add $SUBM_JSWITCH ovn_cluster_sw_submariner_port
ovn_nbctl lsp-set-options ovn_cluster_sw_submariner_port router-port=ovn_cluster_submariner_port
ovn_nbctl lsp-set-type ovn_cluster_sw_submariner_port router
ovn_nbctl lsp-set-addresses ovn_cluster_sw_submariner_port router


ovn_nbctl lr-route-add submariner_router 10.128.0.0/14 $CLUS_JOIN_PORT_IP_A
ovn_nbctl lr-route-add submariner_router 172.30.0.0/16 $CLUS_JOIN_PORT_IP_A
ovn_nbctl lr-route-add submariner_router 172.31.0.0/16 169.254.33.1
ovn_nbctl lr-route-add submariner_router 10.132.0.0/14 169.254.33.1

# steer traffic to cluster B, passing through ovn_cluster_router via the submariner_router
ovn_nbsh
  ovn-nbctl  lr-policy-add ovn_cluster_router 10 "ip4.dst == 172.31.0.0/16" reroute 169.254.34.1
  ovn-nbctl  lr-policy-add ovn_cluster_router 10 "ip4.dst == 10.132.0.0/14" reroute 169.254.34.1
  exit

# disable the operator, and the route-agents, to avoid interferences...
kubectl scale deployment/submariner-operator -n submariner-operator --replicas=0

ROUTEAGENTS=$(kubectl get pods -n submariner-operator  -o jsonpath="{.items[*].metadata.name}" -l app=submariner-routeagent)

for x in $ROUTEAGENTS; do
   kubectl exec -n submariner-operator $x ip l del vx-submariner
done

kubectl delete Daemonset submariner-routeagent -n submariner-operator

# steer traffic from ClusterB arriving ClusterA through the submariner_router external port in br-local
./gwrun.sh # run inside the submariner gateway
# there is still something off… sometimes I see pings working, sometimes not...
    iptables -I FORWARD -i ovn-k8s-gw0 -s 10.128.0.0/14 -d 10.132.0.0/14 -o ens5        -j ACCEPT
    iptables -I FORWARD -i ovn-k8s-gw0 -s 10.128.0.0/14 -d 172.31.0.0/16 -o ens5        -j ACCEPT
    iptables -I FORWARD -i ens5 -s 10.132.0.0/14        -d 10.128.0.0/14 -o ovn-k8s-gw0 -j ACCEPT
    iptables -I FORWARD -i ens5 -s 10.132.0.0/14        -d 172.30.0.0/16 -o ovn-k8s-gw0 -j ACCEPT


    ip rule add from 10.132.0.0/14 to 10.128.0.0/14 table 5
    ip rule add from 10.132.0.0/14 to 172.30.0.0/14 table 5
    ip route add default via $SUBM_EXT_PORT_IP_A table 5
```

#### Alternatives

* eBPF/XDP was explored as an alternative to the current proposal (for the OVN interface),
it was discarded because, while eBPF provides a very powerful interface for dataplane
packet processing, when used on top of an existing solution (OVN/OVS/iptables) we found
the following issues:
  * There is no interface to manipulate/work with the existing connection tracking entries
    on the host.
  * We needed to design a whole network-plugin in parallel to the existing network plugin
    for traffic going to the remote clusters, and traffic coming back.
  * We needed to implement NetworkPolicies manually on top.
  * eBPF would not work on Windows hosts, while, OVN-Kubernetes can work on Windows hosts.

A more detailed view can be found in:
[eBPF/XDP analysys](https://docs.google.com/document/d/1KcrxZROb3v-1EW8B-lOKls4GNwWl73Zg02LytNEbuck)

#### Source code location

Source code should go into the `submariner` repository, the `submariner-routeagent` will
implement the described new driver architecture, in the OVN case handling the gateway
routing rules, as well as the routing entries in all hosts to allow hostNetworking to
remote clusters. The new binary/image `submariner-network-plugin-sync` will be responsible
of database/object synchronization with the existing network plugin.

#### Work items

* [ ] Define an initial architecture interface for the network plugin drivers
* [ ] Improve the submariner-operator to pass down the network-plugin type down
      to the route-agent.
* [ ] Refactor the `submariner-route-agent` event handling into a library which calls the
      new driver architecture and can be used in other similar microservices.
* [ ] Create the `submariner-network-plugin-sync` binary and images,
* [ ] Improve the submariner-operator to create a separate deployment for
      `submariner-network-plugin-sync` where the network plugin driver requires it,
      and pass down the type of network-plugin to the `submariner-network-plugin-sync`
      as well as the `submariner-route-agent`.
* [ ] Converting the existing implementation into the "kube-proxy iptables" which
      will become the default driver. There will be no migration path necessary, or user
      side changes, just internal code architecture changes. (Refine the initially
      defined interfaces/events during this step as necessary)
* [ ] Shipyard support for deploying OVN / multiple network plugins in kind.
* [ ] Program the integration to OVN: connection to NBDB (& SBDB ?), no changes necessary
      to OVN-Kubernetes or OVN have been identified so far. This step is about
      gathering and setting up the proper credentials to connect to NBDB/SBDB as necessay,
      and deciding on cmdline vs go interface.
      Potentially we could use [go-ovn](https://github.com/eBay/go-ovn) and add support
      for routing policies in the library.
* [ ] Program the integration to OVN: Handling routing rules , tables etc on
      gateway nodes & worker nodes. (`submariner-route-agent` driver for OVNKubernetes)
* [ ] Globalnet support with OVN
* [ ] Make any necessary documentation updates across the board: OVN & kube-proxy iptables.
