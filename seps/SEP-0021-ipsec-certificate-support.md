---
sep: 21
title: "IPSec Certificate Support"
status: done
authors: []
created: ""
components: [submariner, subctl]
repos: []
---

# Submariner Enhancement Proposal: Certificate Mode for OVN-Kubernetes IPsec

## Summary

This enhancement enables Submariner to **share the Pluto (Libreswan) process** with OpenShift/OVN-Kubernetes when OVN
IPsec is enabled. The primary goal is to allow both Submariner and OVN-Kubernetes to provide secure, encrypted inter-cluster
and intra-cluster networking on the same OpenShift cluster, without process conflicts or network disruption.

To achieve this, Submariner reuses the existing Pluto process started by OVN-Kubernetes, rather than starting its own.
Because OVN-Kubernetes configures Pluto to use X.509 certificates for authentication, Submariner must also operate in
**certificate mode** as a necessary side effect.

To achieve this, Submariner is enhanced to:

- Reuse the existing Pluto (Libreswan) process started by OVN-Kubernetes, rather than starting its own.
- Operate in certificate mode, matching OVN’s use of X.509 certificates for authentication.
- Provide an explicit deployment option to enable this mode, ensuring that Submariner and OVN-Kubernetes can both manage
IPsec tunnels independently but harmoniously.

---

## Motivation

- **Problem:**
  When both Submariner and OVN-Kubernetes IPsec are enabled, each tries to start its own Pluto process, causing conflicts
and breaking cluster networking.
- **Goal:**
  Allow Submariner and OVN-Kubernetes to run together, each managing their own tunnels, but sharing
the same Pluto process and certificate-based authentication infrastructure.

---

## Design Details

### Manual Enablement

- Certificate mode must be **explicitly enabled** by passing an option (e.g., `--ipsec-cert-mode`) to `subctl` during
deployment or join.

### Pluto Process Expectation

- When certificate mode is enabled, Submariner **expects** that a compatible Pluto process (as started by OVN-Kubernetes)
is already running
- Submariner does **not** attempt to start or manage the Pluto process in this mode. It only writes its connection
configuration (`/etc/ipsec.d/submariner.conf`) and loads connections into the existing Pluto instance.
- If the Pluto process is not running when Submariner is deployed in certificate mode, tunnel setup will fail and errors
will be logged.

### Certificate Management

- Submariner automates certificate issuance, signing, and import into the NSS database (used by Libreswan for authentication),
using Kubernetes Secrets and controllers.
- The NSS database used by Submariner for certificate storage is located at `/var/lib/ipsec/nss` on the Gateway node.
- The CA is managed in the broker cluster, and certificate Secrets are synchronized between broker and local clusters.
- **Certificate Signing Flow:**
  - The Submariner Gateway generates a Certificate Signing Request (CSR) and sends it to the broker cluster.
  - The broker cluster, via the operator pod, approves and signs the CSR.
  - The signed certificate, along with the CA certificate, is then copied back to the requesting cluster from the operator pod in the broker.
  - The signed certificate and CA are imported into the local NSS database, enabling secure certificate-based authentication for IPsec tunnels.

### RBAC and Cleanup

- Proper RBAC is set up for certificate management, and uninstall logic ensures clean removal of Submariner’s configuration and certificates.

### Status and Observability

- Connection status is reported in the Gateway object, the existing logic does not require any changes for this.
- Debug logs are added for certificate import and NSS operations.

---

## Usage

- The administrator must pass the appropriate option to `subctl`:

  ```sh
  subctl join --ipsec-cert-mode broker-info.subm
  ```

- This will configure Submariner to:
  - Use certificate-based authentication.
  - Reuse the existing Pluto process.
  - Avoid any actions that would interfere with OVN-Kubernetes IPsec.
- It is the administrator’s responsibility to ensure that the Pluto process is running and managed by OVN-Kubernetes before deploying Submariner
in this mode.
