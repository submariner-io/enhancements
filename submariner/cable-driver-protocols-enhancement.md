# Cable Driver Enhancement - IP-in-IP

## Summary

Submariner today supports three cable drivers: VXLAN, IPSec (tunnel mode) and WireGuard. The
goal of this proposal is to outline additional enhancements needed for the Submariner Cable Driver
to support of IP-in-IP for unencrypted connections between clusters or for use as an overlay with
IPSec in transport mode for encrypted connections between clusters.

For IP in IP, the benefits fall into the following categories:

* **Performance**: IP in IP is has a lower overhead than VXLAN which could lead to performance
improvements in terms of packet overhead (bandwidth consumption) as well as the cost of processing
packets. Some of the performance improvements are shown below [1]:

  * ![Throughput rate vs Kernel](./images/tx-tput-rate.png)
  * ![Throughput rate Gbps](./images/tx-tput-gbps.png)
  * ![CPU cycles](./images/cpu-cycles.png)

* **Interoperability**: IP in IP can be used to transport packets between domains when the protocol
in those domains (e.g. IPv6) is not supported by intermediary networks (e.g. IPv4).

## Proposal

This enhancement proposes to support IP-in-IP for un-encrypted connection between clusters. A
new IP tunnelling interface `ipip-tunnel` will be added to the active gateway node of each cluster
and it will act as the Tunnel End Point. The traffic which is destined to other clusters will be
forwarded via this interface to the appropriate clusters.

> **_NOTE:_** A single IP Tunnelling interface will be added, and tunnels managed using
`# ip route add encap ip <args>` in order to avoid a point to point connection per tunnel.

### Design Details

* A new interface "ipip-tunnel" will be added.
* The IP tunnelling interface will be one-to-many, that is a single interface will be used to
connect to all the joined clusters.
* For the "ipip-tunnel" IP, the prefix "243" will be used, followed by the rest of the blocks
from the endpoint private IP address. For example, if the private IP address is 10.2.96.1 the
"ipip-tunnel" IP would be 243.2.96.1.
* The routes will be added to forward all the Service and Pod CIDR traffic to the respective
remote Tunnel IPs.
* Table 150 shall be used to add the routes for inter-cluster communication.

Example of routeing rules added by the IPTun driver on a gateway node to other nodes

```console
ip route add 10.2.0.0/16 encap ip id 100 dst 172.18.0.21 via 243.18.0.21 dev ipip0 table 100 metric 100 src 10.1.96.0
ip route add 100.2.0.0/16 encap ip id 100 dst 172.18.0.21 via 243.18.0.21 dev ipip0 table 100 metric 100 src 10.1.96.0
ip route add 10.3.0.0/16 encap ip id 100 dst 172.18.0.8 via 243.18.0.8 dev ipip0 table 100 metric 100 src 10.1.96.0
ip route add 100.3.0.0/16 encap ip id 100 dst 172.18.0.8 via 243.18.0.8 dev ipip0 table 100 metric 100 src 10.1.96.0
ip route add 10.4.0.0/16 encap ip id 100 dst 172.18.0.4 via 243.18.0.4 dev ipip0 table 100 metric 100 src 10.1.96.0
ip route add 100.4.0.0/16 encap ip id 100 dst 172.18.0.4 via 243.18.0.4 dev ipip0 table 100 metric 100 src 10.1.96.0
```

and the resulting routes on the gateway node:

```console
[root@cluster1-worker submariner]# ip route show table 100
10.2.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.8 ttl 0 tos 0 via 243.18.0.8 dev ipip-tunnel src 10.1.160.0 metric 100
10.3.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.7 ttl 0 tos 0 via 243.18.0.7 dev ipip-tunnel src 10.1.160.0 metric 100
10.4.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.10 ttl 0 tos 0 via 243.18.0.10 dev ipip-tunnel src 10.1.160.0 metric 100
100.2.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.8 ttl 0 tos 0 via 243.18.0.8 dev ipip-tunnel src 10.1.160.0 metric 100
100.3.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.7 ttl 0 tos 0 via 243.18.0.7 dev ipip-tunnel src 10.1.160.0 metric 100
100.4.0.0/16  encap ip id 100 src 0.0.0.0 dst 172.18.0.10 ttl 0 tos 0 via 243.18.0.10 dev ipip-tunnel src 10.1.160.0 metric 100
```

A high level view of the topology is shown in the diagram below:
![IP-in-IP Tunnelled Gateway](./images/ipip_cable.png)

> **_NOTE:_** IP-in-IP has a lower processing and bandwidth overhead than VXLAN.

## References

[1] Ryo NAKAMURA, Improving Packet Transport in Virtual Networking by Encapsulation Techniques
<https://repository.dl.itc.u-tokyo.ac.jp/record/51086/file_preview/A34115.pdf>
