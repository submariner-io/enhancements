# IPSec Certificate Issuer

## Summary

Submariners current security arrangements, namely using an identical pre-shared key in
all deployments will fail security assessment in most environments which have to comply
with a pre-existing security policy and rules on using VPNs in general and IPSEC in particular.

In order to comply the following minimum enhancement that needs to be implemented is the use
of X509 certificates per Gateway. This will require the introduction of a Certificate Authority
(CA) that's capable of signing Certificate Signing Requests (CSRs) for Gateways in the
`ClusterSet`. This CA will be serviced by a Syncer that will act as a proxy between the Certificate
Requester and the CA itself.

## Proposal

This enhancement proposes to use a `ClusterSet` wide `Certificate Authority` and a `Syncer`
that is co-located with the Broker to service Gateway. At a high level this requires:

* A Certificate Authority that can be used across the ClusterSet. Ideally this authority resides
on the Broker node.
* A CA_CSR_Syncer on the Broker Node.
* A new CRD to encapsulate a Submariner CSR (towards this centralized CA), and the result (towards
the Gateway). Gateways will fill the CRD with relevant information (a base64 encoded string of a PEM
encoded certificate request) and publish the CR to the Kube API.
* A GW_CSR_Syncer on the gateway Node.

The general idea is that, when a Broker is installed, a CA and CA_CSR_Syncer are also installed on
the Broker node. Then, when a Submariner gateway starts up it will launch a GW_CSR_Syncer, it will
also create a CSR, base64 encode it and send it via a Submariner CSR CRD towards the Broker node CA.
When the CA_CSR_Syncer picks up the Submariner CSR resource, it will issue a CA CSR towards the CA.
Monitor the result of the CA CSR and publish the result back to the original Gateway via the Broker
(by updating the original CSR CRD). The details of the design are shown in the upcoming sections.

## Design Details

### CA Deployment

The following diagram shows the various component needed for the central CA.

![CA Deployment](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/maryamtahhan/enhancements/feat_central_cert_manager/submariner/images/CA-deploy.puml)

### CA Issue Certificate Example

> **_NOTE:_**
>
> * For Simplicity purposes the role of Admiral in the diagrams below has been
> collapsed into the CSR_Syncers.

![CA Issue Certificate Example](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/maryamtahhan/enhancements/feat_central_cert_manager/submariner/images/CA-operation.puml)

### Submariner CSR CRD

```Go
type SubmarinerCSRs struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    // CSR towards the Submariner CA.
    CSRSpec SubmarinerCSRSpec `json:"spec"`

    // Result of CSR.
    CSRStatusSpec SubmarinerCSRStatus `json:"spec"`
}

// SubmarinerCSRSpec defines the CSR for the Submariner Gateway
type SubmarinerCSRSpec struct {
    // The requested 'duration' (i.e. lifetime) of the Certificate.
    Duration *metav1.Duration

    // The PEM-encoded x509 certificate signing request to be submitted to the
    // CA for signing.
    Request []byte

    // Usages is the set of x509 usages that are requested for the certificate.
    // "signing", "digital signature", "key agreement"...
    // details can be found here: https://www.rfc-editor.org/rfc/rfc5280#section-4.2.1.3
    // and https://github.com/cert-manager/cert-manager/blob/master/internal/apis/certmanager/types.go#L159
    Usages [] string
}

// SubmarinerCSRStatus defines the status of CertificateRequest and
// resulting signed certificate.
type SubmarinerCSRStatus struct {
    // Status of the request (`Ready`, `InvalidRequest`, `Approved`, `Denied`).
    Status string

    // The PEM encoded x509 certificate resulting from the certificate
    // signing request. Empty if the Status is InvalidRequest/Denied.
    Certificate []byte

    // The PEM encoded x509 certificate of the Certificate Authority.
    CA []byte
}
```

### CA CSR CRD

The Proposed CA project to leverage is [cert-manager](https://cert-manager.io/docs/) Here is an
example of the [CA CSR](https://cert-manager.io/docs/concepts/certificaterequest/) and also [here](https://github.com/cert-manager/cert-manager/blob/master/internal/apis/certmanager/v1alpha2/types_certificaterequest.go#L56)

On the Broker the CA_CSR_Syncer will:

* Read incoming Submariner CSR resources from Gateways.
* Translate the Submariner CRs to CA CSR resources.
* Submit the CA CSR to the CA.
* Read the resulting certificate request that contains the signed Certificate.
* Update the Submariner CSR resource with the Signed Certificate (alternatively - this could be a
separate Certificate CRD).
* Propagate the Signed Certificate back to the originating GW (leveraging labels and
annotations) using the Broker.

CA installation and Issuer configuration details can be found here: https://cert-manager.io/docs/installation/
and https://cert-manager.io/docs/configuration/ca/