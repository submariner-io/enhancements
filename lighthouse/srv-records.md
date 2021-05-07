# SRV records in Lighthouse

## Summary

Lighthouse only supports A record, but there are use case where we support for other types records is required

## Proposal

This enhancement proposes to add support for SRV records in Lighthouse. The support will be added for both ClusterIP Service
and Headless Service. The proposal only covers the use case where the service ports have the same name protocol and port number across the exported
services with the same name and namespace.

## Design Details

The SRV records of the below formats shall be supported.

### Cluster IP Service

- \<portname\>.\<protocol\>.\<svc-name\>.\<svc-namespace\>.svc.clusterset.local -> will return the SRV record with the port
number and domain name with matching port name and protocol from one of the joined clusters where the service is available.
The clusters will be selected based on the Round Robin algorithm.

- \<portname\>.\<protocol\>.**\<cluster-name\>**.\<svc-name\>.\<svc-namespace\>.svc.clusterset.local -> will return SRV record
from the cluster specified, if available with matching portname and protocol

If \<port-name\> and \<protocol\> is not specified one SRV record will be returned for each port configured for the service in
the selected/specified cluster.

### Headless Service

- \<portname\>.\<protocol\>.\<svc-name\>.\<svc-namespace\>.svc.clusterset.local -> will return the SRV records  with port number
and domain name for each backend pod with matching portname and protocol in every joined cluster where the service is available.

- \<portname\>.\<protocol\>.**\<cluster-name\>**.\<svc-name\>.\<svc-namespace\>.svc.clusterset.local -> will return the SRV records
with port number and domain name for each backend pod with matching portname and protocol, in the specified cluster.

If \<port-name\> and \<protocol\> is not specified one SRV record will be returned for each port configured for the service in the selected
cluster or all clusters based on the query.

## Work items

- [Enhancement proposal](#525)
- [Support for SRV records](#526)
- [Add e2e](#527)
- [Update Docs](https://github.com/submariner-io/submariner-website/issues/510)
