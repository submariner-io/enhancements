# IPSec Certificate Issuer

## Summary

Submariner's current security arrangements, namely using an identical pre-shared key in
all clusters in a ClusterSet, will fail security assessment in most environments which have to comply
with a pre-existing security policy and rules on using VPNs in general and IPSEC in particular.

In order to comply, the minimum enhancement that needs to be implemented is the use
of X.509 certificates per Gateway. This will require the introduction of a Certificate Authority
(CA) that's capable of signing Certificate Signing Requests (CSRs) for Gateways in the
`ClusterSet`. This CA, a set of controllers (one to authorize and another to sign CSRs),
will be serviced by a Syncer that will act as a proxy between the Certificate requester and
the CA itself.

## Proposal

This enhancement proposes to use a `ClusterSet`-wide `Certificate Authority` that is co-located
with the Broker to service Gateway CSRs. The `Certificate Authority` consists of two controllers:

- A CSR `Approval controller` that checks the identity of the CSR requester and
- A CSR `Signing controller` that signs approved CSRs. The signerName for Gateway CSRs will be
`submariner.com/broker-signer`.

A `Syncer` is also needed to synchronize CSRs to/from the Broker.

## Design Details

A root certificate is generated for the `Certificate Authority` signer that gets passed to all the
clusters. The Broker `ca.crt` will be reused as the Signers root certificate.

At Gateway startup, the Gateway will install the central CA cert, generate a private key and a CSR
with the `signerName: submariner.com/broker-signer`.

If the request is from a remote node (to the Broker), a local Syncer will synchronize the CSR to the Broker
node. The Approval Controller verifites that the request is from a valid cluster in the ClusterSet and the
Signing controller signs the authorized CSR and updates the `status.certificate` Field. The Syncer on the
Broker node will then propagate the issued CSR back to the originating cluster. The Syncer on the originating
cluster will pull in the new Certificate for the Gateway and verify the signer. The Gateway then uses that
certificate as part of its IPSec Configuration.

## References

- [CSR Approving Controller](https://github.com/open-cluster-management-io/addon-framework/blob/0379f4992da025688d78023fafa7a11c07be4714/pkg/addonmanager/controllers/certificate/csrapprove.go#L39)
- [CSR Signing Controller](https://github.com/open-cluster-management-io/addon-framework/blob/0379f4992da025688d78023fafa7a11c07be4714/pkg/addonmanager/controllers/certificate/csrsign.go)
- [Example Approver and Signer Functions](https://github.com/open-cluster-management-io/addon-framework/blob/f2fc48e63c4cdadcce3881be148ee10bf85c6fb7/pkg/utils/csr_helpers.go#L29)
