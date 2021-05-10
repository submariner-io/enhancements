# Encryption-less connections between clusters with VXLAN

## Summary

All current cable drivers involve encrypting content, which is great for privacy but involves some overhead.
On private connections, it would be useful to allow unencrypted connections, for example using IP-on-IP or VXLAN.

## Proposal

This enhancement proposes to use VXLAN for un-encrypted connection between clusters. A new VXLAN interface "vxlan-tunnel"
will be added to the active gateway node of each cluster and it will act as the VTEP. The traffic which is destined to other
clusters will be forwarded via this interface to the appropriate clusters.

## Design Details

* A new interface "vxlan-tunnel" will be added.
* The VXLAN interface will be one-to-many, that is a single interface will be used to connect to all the joined clusters.
* For the VXLAN VTEP IP, the prefix "241" will be used, followed by the rest of the blocks from the endpoint private IP address. For example,
if the private IP address is 10.2.96.1 the VTEP IP would be 241.2.96.1.
* The remote cluster endpoint IPs (either public or private) will be added to the Forwarding Database.
* The routes will be added to forward all the Service and Pod CIDR traffic to the respective remote VTEP IP.
* Table 150 shall be used to add the routes, which is otherwise used for host network traffic routes. We do not require any
extra rules to handle host network traffic for the VXLAN driver.
* The IPSEC_NATT_PORT used port shall be used as the VTEP port. The parameter name shall be changed to TUNNEL_PORT. This requires changes
in Shipyard, Submariner-Operator, Submariner-Website, Helm and Submariner

## Work items

* [New cable driver](https://github.com/submariner-io/submariner/issues/1149)
* [CI](https://github.com/submariner-io/submariner/issues/1148)
* [Deployment(Operator, subctl)](https://github.com/submariner-io/submariner-operator/issues/1101)
* [Deployment (helm)](https://github.com/submariner-io/submariner-charts/issues/116)
* [Metrics](https://github.com/submariner-io/submariner/issues/1150)
* [Docs](https://github.com/submariner-io/submariner-website/issues/450)
