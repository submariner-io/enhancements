# Submariner Broker Multi-Tenancy Support

Extending Submariner Broker to Support Multiple ClusterSets/Tenants.

## Summary

Currently, Submariner Broker supports a single ClusterSet (i.e., a set of
clusters with a high degree of trust and sharing). This design document
proposes an enhancement to support multiple disjoint ClusterSet’s (e.g.,
each belonging to a different “tenant”) hosted on a single Broker cluster.
 The design addresses multiplicity and isolation concerns.

## Motivation

In SaaS offerings, such as managed OpenShift, it may be desirable to
minimize the cost of running a dedicated Broker cluster for each ClusterSet.
When all clusters share a high level of trust, the Broker can be co-located
with workloads and does not require a dedicated cluster. However, this level
of trust is neither uncommon nor desirable in a multi-tenant environment.
Furthermore, in such settings, the Submariner Broker cluster is owned
and managed independently of the workload clusters.

Submariner repository currently has the following related issues:

- [submariner#1120](https://github.com/submariner-io/submariner/issues/1120):
Deploy multiple ClusterSet's within the same "Submariner"
- [submariner-operator#1040](https://github.com/submariner-io/submariner-operator/issues/1040):
Support deployment within a custom namespace

In addition, a recent PR which provides partial support was recently merged:

- [submariner-operator PR 1682](https://github.com/submariner-io/submariner-operator/pull/1682)

### Pitfalls of the current state

The current design places all resources relating to a ClusterSet in a single,
predefined, namespace on the Broker cluster. This works well for a single
ClusterSet and users often choose to run independent Submariner Broker clusters
for different ClusterSets. The use of a single predefined Broker namespace
is limiting:

1. The selected namespace name should be configurable to accommodate
requirements of different use cases.
2. It prevents Submariner from managing multiple independent ClusterSets in a
single Broker cluster.

In a SaaS environment, we can consider three distinct user types in the system,
with different privileges:

- **Offering Admin** owns and manages the Submariner Broker(s) cluster. They are
responsible for creating a per ClusterSet configuration space.
- **ClusterSet/Broker Admin** manages workload clusters belonging to a ClusterSet.
They manage cluster membership in a given ClusterSet. Admins of different ClusterSets
should be isolated from each other.
- **Cluster/Service Admin** manages the import and export of services from/to
workload clusters in the ClusterSet.

The first two are somewhat conflated in the current Submariner design.

### Goals

- Allow usage of configurable Broker namespace names.
- Enable multiple ClusterSets to coexist in the same Submariner Broker(s)
cluster.
- Provide isolation (via namespaces and RBAC) between objects of different
ClusterSets to ensure users can’t modify configuration of another user’s ClusterSet.
- Provide clearer distinction between Offering Admin and ClusterSet Admin,
by allowing the latter to deploy a Broker with restricted namespace permission.
- Provide backward compatibility for existing deployments (e.g.,
explicit opt-in, defaulting to current Submariner behavior and settings by default).

### Non-Goals

- Support live migration of an existing Broker to support multi-tenancy.
- Support for workload clusters joining more than one ClusterSet
(in the case of multiple tenants in workload clusters, we could run a
separate cluster control plane per tenant).
- Any changes needed in runtime and data plane elements (e.g., metrics collection,
 Lighthouse, etc.) to support configurable ClusterSet naming.
- Support multi-tenancy in workload clusters.

Note that non-goals are defined as such in the context of the currently proposed
work and may be added in the future. They are optional for the current phase
and may be implemented if straightforward.

## Proposal

The work will be carried out in stages:

1. Make Broker namespace configurable (e.g. in CRDs and subctl) to
allow multiple Broker namespaces to co-exist. The default namespace value
can match the current fixed value to allow migration. The use of multiple
or single namespace should be under global configuration to allow backwards
compatibility. The change does not introduce new functionality (e.g.,
isolation and protection) beyond configurable namespace names. This allows
for multiple namespaces to exist concurrently, in a cooperative manner
(i.e., discretionary access control by referencing the correct namespace).
2. Support stricter isolation between namespaces. This step builds upon the
previous and creates relevant service accounts, RBAC policies, etc., to ensure
users can only access their designated Broker namespace (previous step assumed
cooperative access).
3. Separate Broker namespace creation and management to separate the Offering
and ClusterSet Admin roles and privileges. This could be done via a Broker (full)
 reconciliation loop (note: currently the broker is not active, but acts
 as a storage namespace only). Changes should be made to minimize disruption to existing
workflows (e.g., hide changes behind the existing `subctl` command line).

The first and second steps are mostly complete by
[submariner-operator PR 1682](https://github.com/submariner-io/submariner-operator/pull/1682).

## Design Details

### Overview of Proposed Changes

- Add a new “central” controller/operator called Axon controller (may think
of a better name.) to the Broker cluster. The controller runs in a new,
singleton, Axon namespace. This controller is responsible for provisioning
submariner tenants (i.e., a ClusterSet). The `subctl` CLI will be extended
to support tenant management by adding the following commands (see details below):
  - `subctl deploy axon-cp`
  - `subctl create|delete|get|list tenant [tenant_id]`
- Create the submariner CRDs when deploying the Axon operator, assuming that
tenants (i.e., ClusterSet Admins) will not have permissions to create those.
The `deploy broker` command will ensure the CRDs exist.
- A submariner tenant is represented by a tenant object in the Axon namespace
 and a dedicated tenant namespace. Each tenant has a string ID and the namespace
 is some constant prefix + tenant ID.
- A SA is created per tenant and a *partial* `broker-info.subm` file is generated
 by `subctl` upon tenant creation, with credentials (client secret) to access
 the tenant namespace only.
- Tenant can be deleted by deleting the tenant object, and the Axon operator
 will clean up its resources (i.e., remove namespace and all contained objects)
- The `deploy broker` command of subctl will require some changes but will
 also maintain backwards compatibility with the single broker operation mode:
  - Add an optional `namespace` field to the `.subm` file, to identify the
  tenant. If the field exists, it takes precedence over the `--namespace` CLI
  argument. If neither is provided, uses the default broker namespace.
  - Use the access credentials and the Broker URL that are already specified in
  the `.subm` file to access the Broker, if they exist. Otherwise - fallback to
  details from kubecontext.
  - The PSK will still be generated only in the `deploy broker` command, and will
  be added to the `.subm` file (backing-up the old file).
  - Deprecate the `--ipsec-psk-from` CLI option, as its name is not representative
  anymore, and using `--broker-info-from` option as a synonym.
  - The submariner operator in the Broker cluster will not be deployed anymore,
  as it is not doing anything currently, and in Axon mode it cannot be deployed to
  the submariner-operator NS in broker due to namespace-restricted privileges. Our
  insight is that in the future, we may deploy a broker-specific operator per
  tenant in the tenant namespace, but it will require some changes, e.g. not
  creating non-broker CRDs and other objects.
    - An alternative is to still deploy the submariner operator in the broker, but
    always in the tenant namespace. This means that two operators will exist if the
    cluster is operating in mixed mode (i.e., both as a Broker and a worker).
- The tenant management and `deploy axon-cp` commands shall run by an Offering
Admin with a role of Kube admin. The `deploy broker` command shall run by the
ClusterSet Admin with (at least) a broker admin role, which is created per tenant.
It is also possible to create an axon-admin role, so that tenant management is
carried-on with this role, and only `deploy axon-cp` runs as Kube admin.

### Command Flow

Workflow in each new and changed command is described below.

---

`subctl deploy axon-cp`

Shall be executed by Offering admin.

1. Get target cluster from kube context
1. Deploy `axon-operator` (consider a better name)
    1. Ensure (create if missing) NS exist: `axon-operator`
    1. Ensure CRD exist: `tenants.submariner.io`
    1. Ensure CRDs relevant to submariner broker exist:
        - brokers
        - clusters
        - endpoints
        - serviceimports
        - serviceexports  
    1. Setup RBAC: Ensure `axon-operator` SA, Role, RB exist
    1. Optional: setup RBAC for axon-admin
    1. Ensure axon-operator Deployment is created in `axon-operator` NS

---

`subctl create tenant [tenant_id]`

Shall be executed by Offering admin.

1. Generate a GUID for `tenant_ID` if it is empty. Print tenant ID to stdout, so
it can be tracked in case of CLI termination prior to completion.
1. Put a `tenants.submariner.io/tenant` object with name `tenant_id` in NS `axon-operator`.
    - Fail and terminate CLI if an object with such name already exists.
    - Note: provisioning UI may also use GUIDs to avoid name collisions. It will
    also probably manage additional tenant data like the “owner” user etc.
    Still the responsibility for naming the tenant is on the client, and
    this makes it easier to track orphaned objects in case of failures.
1. Axon operator watches the `axon-operator` NS and gets notified of the new
Tenant object (`subctl` blocks meanwhile)
1. Axon operator changes tenant object status to “Provisioning”
1. Axon operator ensures namespace `submariner-broker-<tenant_id>` exists
1. RBAC: Axon operator ensures (or creates) `submariner-k8s-broker-admin`
SA, Role, RB exist in NS `submariner-broker-<tenant_id>`. The role is created
with limited permissions: mutating Broker-related CRD objects only.
1. Axon operator set status in tenant object to “Provisioned”, update creation
timestamp, etc.
    - On any failure, Axon operator set tenant object status to “Failed” instead.
1. `subctl` get notified for tenant object status change
    - If status is “Failed” - exit CLI with error.
1. `subctl` writes a broker info file `broker-info.subm`:
    1. Set `namespace` (new field) = `submariner-broker-<tenant_id>`
    1. Read client secret token from `submariner-k8s-broker-admin` SA
    1. Read broker URL from current k8s context
    1. Write `.subm` file to disk
    1. Note: Consider uniquely naming `.subm` file: `broker-info-<tenant_id>-timestamp.subm`

---

`subctl delete tenant <tenant_id>`

Shall be executed by Offering admin.

1. Delete tenant object with name `tenant_id`
1. Axon operator gets notified of the deletion
1. Axon operator ensures namespace `submariner-broker-<tenant_id>` and all nested
objects do not exist (remove if necessary)
1. Axon operator blocks until namespace does not exist, fails on timeout.

---

`subctl get tenant <tenant_id> [--generate_subm]`

Shall be executed by Offering admin.

- Shows status and creation time for a tenant.
- Optionally also write a broker info file for a provisioned tenant -
step 9 in create tenant. `subctl` shall make sure tenant status is Provisioned
before writing the file.

---

`subctl list tenants`

Shall be executed by Offering admin.

- Shows all tenants and their status.

---

`subctl deploy broker`

Shall be executed by ClusterSet (Broker) admin.

Changes are highlighted in italics.

1. *If `broker-info.subm` file specified (`--broker-info-from` or the deprecated
`--ipsec-psk-from`), set following context pieces from file in case they exist*:
    - Broker Namespace
    - User (`submariner-k8s-broker-admin`)
    - Broker cluster URL
1. *Fallback to non-axon mode connection details*:
    1. In case namespace is not set, take from `--namespace` arg or fallback to
    default NS.
    1. In case user is not set, fallback to kube context
    1. In case Broker cluster URL is not set, fallback to URL from kube context
1. Ensure (or create) broker namespace exist
1. Setup RBAC
    1. Ensure broker admin SA `submariner-k8s-broker-admin` exists in broker NS.
    1. Ensure broker admin role exists in broker NS `submariner-k8s-broker-admin`.
    *Ensure it has the expected permissions to mutate submariner broker-related CRs.
    Fail CLI if unable to mutate permissions to the expected set.*
    1. Ensure role binding exists in broker NS (SA + role)
    1. Backward compatibility - create legacy RBAC:
        1. *Skip this in case `tenants.submariner.io` CRD exists - this means we
        are in Axon mode, and we do not need to support legacy RBAC (and also cannot
        create a cluster role).*
        1. Create broker client service account in NS if not exist (`submariner-k8s-broker-client`)
        1. Create broker cluster cluster-role in broker NS (`submariner-k8s-broker-cluster`)
        1. Create role binding
    1. Wait for broker admin SA secret token to be ready
1. *Deploy operator (deleted, will not be carried on anymore)*
1. Deploy broker
    - Create or replace broker object in broker NS with spec parameters
    from `subctl` CLI
1. Write broker info local file (`broker-info.subm`)
    1. Backup `.subm` file if already exists
    1. Construct `.subm` file state
        1. Read client secret token from `submariner-k8s-broker-admin` SA
        1. Read ipsec PSK from old `.subm` file or generate a new PSK.
        1. Read broker URL from current k8s context current URL in use
        1. *Set `namespace` field to broker namespace.*
        1. Update `.subm` file state from CLI args: components, service discovery,
        custom domains.
        1. Validate existing global networks
        1. Create globalnet config map
        1. Write `.subm` file to disk

## Alternatives

- Running multiple brokers (e.g., dedicated clusters or kcp/k3s based) is another
option. This incurs management and resource overheads. Future implementation could
use nested API masters (e.g., kcp/k3s) inside the main Broker cluster for better
isolation, as long as a management framework is in place to allow automated scaling
(e.g,. CRD based).
- Multi-tenancy as the default behavior: `subctl create tenant` also deploys the
broker (by reconciliation in the axon operator) and the `subctl deploy broker`
 command is merged into it. This will probably involve further changes and will
 possibly break some backward compatibility.
- Tenant management operations can be extracted to a separate CLI tool.
E.g. `axonctl`.

## Open Questions

- Do we need to update helm too?
- Do we still need to support the `--namespace` CLI arg? Is it still required to
support multiple ClusterSets in classic (non-Axon) submariner?
- How do we authenticate and authorize users beyond the Broker Admin (e.g., k8s
service accounts)? Should we support additional principles (e.g., users internal
to a tenant) and permissions within a ClusterSet?
- Do we create a new repo for axon-cp operator? Or put it in
submariner-operator repo?
