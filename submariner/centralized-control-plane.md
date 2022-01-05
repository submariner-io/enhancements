# Centralized Service Control Plane

## Summary

Current Submariner service management follows the model and guidelines
 defined in [MCS](https://github.com/kubernetes-sigs/mcs-api). Specifically the
 current design has these attributes:

- Exported services are automatically reflected into all clusters in the ClusterSet;
- Service are exported and imported using the same namespace and name; and
- Service exports are managed from workload clusters.

The above works well in an environment where clusters are used by a single
 administrative domain and services are commonly shared across all clusters.
 For example, when a company runs clusters as the runtime infrastructure and developers can
 deploy to any cluster for availability, redundancy, or geographic proximity. However, it is
 less suited in a deployment model where clusters are owned and operated by different teams,
 each having independent control and configuration (see next section for elaboration).
 For those deployments, we would like to propose a different approach for Service management,
 that allows:

1. Independent service naming (i.e., allow use of different names in different clusters).
1. Selective imports (i.e., import services into a subset of clusters in the ClusterSet).
1. Centralized control over exporting and importing (i.e., defined in Broker, not
  workload clusters).

The design proposal attempts to achieve the above with minimal changes to workload
 clusters, especially with respect to data plane configuration. Ideally, the control
 plane changes proposed will be confined to the Broker cluster only.

## Additional Background and Context

Based on work we've done in Istio (see for example [this](https://istio.io/latest/blog/2019/isolated-clusters/)),
 we identified two common usage models for clusters:

- **Shared Infrastructure**: clusters are (mostly) identically configured (e.g., namespaces,
  RBAC, etc.). Each team is allocated the same set of namespaces across all clusters and can
  deploy to any cluster. The boundary between teams is the namespace. This is the model assumed
  by MCS and Submariner.
- **Federated**: clusters are used as the boundary between teams (e.g., they may even be
  billed to different departments). Each team can only deploy to a subset of the clusters
  (i.e., the ones they "own") and is granted full reign over that cluster. They are free to
  choose whatever names they desire. Thus, the federated mode cannot assume name sameness.
  The teams may still have a shared clusters where the "integrated application" runs, pulling
  in services from the per team clusters.

The federated usage model needs to support the following use cases (*partial list*...):

- I'd like to export my internal services only to my other clusters, but still consume shared
  services (e.g., centralized monitoring and alerting) provided by other teams' clusters.
- I'd like to decouple the local and global names used for services and import a remote service
  into my namespace using a different name (e.g., two teams can have an "authz" service running
  in "frontend" namespace, but each is unique).
- When importing a service I'd like to control its "name" and "location" (e.g., import into a
  specific namespace with a specific name). I can't assume that the same namespaces exist
  across all clusters and importing with the same name may conflict with an existing service.

### Relation to MCS API

Based on the above, it should be clear that in "federated" environments service naming and
 sharing may be controlled differently than proposed by the MCS API. The MCS API objects
 are building blocks (i.e., a mechanism), and as such can be used by a different policy engines
 to derive imports and exports, locally or from a central location. This might be
 "import everywhere" as in Submariner, or a more selective TBD policy.

The MCS API leaves the following aspects out of its scope:

- Name independence across clusters;
- The mechanism to propagate Exports and Import from/to workload clusters.
  A synchronization/distribution mechanism is required (e.g., Submariner's Lighthouse is
  one such example). The mechanism is implementation specific; and
- Policy objects (e.g., how to declaratively define which exports to import where and with
  what local workload clusters attributes). These are similarly implementation specific.

The first aspect (name independence) seems to me a fundamental assumption on the MCS API.
 This makes `ServiceExport` and `ServiceImport` less useful as the abstraction in the central
 shared location. For example, since all imports are created in the Submariner broker namespace
 of the ClusterSet, Submariner resorts to tagging the destination namespace and name on the
 `ServiceImport` using labels and annotations. It might have been clearer to explicitly
 set those in `ServiceImport.Spec` if MCS provided that. Similarly, the assumption on name
 sameness has implication on `ServiceExport` (e.g., two completely independent cluster, owned
 and managed by different teams, must agree on using unique namespaces and/or names, lest their
 exports conflate when shared).

While all three (naming, synchronization and policies) could be considered as possible
 enhancements to the MCS API, it would be worthwhile to experiment with different approaches,
 such as the one defined here, before attempting to reach community consensus on MCS API changes.

## Proposal

### Goals

- Enable independent service name and namespace in workload clusters.
- Enable a Service to be imported into a subset of the workload clusters only.
- Enable centralized control over definition of Service exports and imports. This implies
  disabling, as much as possible, these definitions from taking effect when created
  directly on workload clusters. This should enable a future extension to support
  centralized, policy driven, control over service sharing.
  
### Non-Goals

- Maintain concurrent operation of both existing (distributed) and proposed
  (centralized) service control planes. A ClusterSet would operate exclusively
  in one mode or the other.
- Maintaining `subctl` as the UX is optional. The proposal targets an API
  driven control plane (i.e., CRD based), so adding `subctl` commands can be
  done at a later stage.

## Design Details

### Service Control Plane Objects

The Service object model is built around three new CRDs, defined only in the Broker
 cluster. The new CRDs are used to generate the corresponding MCS CRDs, which are
 then replicated to the workload clusters, as today. An `axon` tag is used as the
 [K8s API `Group`](https://book.kubebuilder.io/cronjob-tutorial/gvks.html)
 to differentiate from the Kubernetes and MCS objects with the same name:

1. `axon:Service` defines a service that can consumed by and/or provided from
  multiple clusters. The Service object represents some API deployed
  in the system. It decouples the local service names used in workload clusters
  from the name that is shared between clusters.
1. `axon:ServiceBinding` creates an association between a local service and the
  global name. The object plays a role somewhat akin to `mcs:ServiceExport` but
  The level of indirection supports breaking the "name sameness" assumption.
1. Similarly, `axon:ServiceImport` plays the role of `mcs:ServiceImport` but allows
  extending the MCS definition, where and when needed.

The following CRDs are proposed to support the new design. The CRD definitions
 below are partial and capture only essential parts needed to illustrate the design.
 For brevity, the standard Kubernetes top level CRD definition (e.g., `TypeMeta`,
 `ObjectMeta`, `Spec` and `Status`) is omitted and left as an exercise for the
 implementor... Similarly, all `Status` types are assumed to contain a `Conditions`
 field of type `[]metav1.Condition`, which is also omitted below. Consequently,
 Status objects containing only this field are also omitted, for brevity.

A reference to Clusters in the ClusterSet is defined by providing
 a unique identifier for each cluster (denoted as a `ID` below. An ID can be
 an arbitrary string, such as a UUID, but can also use more descriptive
 names as long as they're not ambiguous within a ClusterSet).

```Go
// ServiceSpec defines the desired state of a Service
type ServiceSpec struct {
   // ID is a unique (within a ClusterSet scope) identifier for the Service object.
   // Note that the CRD object name could also be used, in which case ID becomes
   // redundant.
   ID string `json:"id,omitempty"`
   // VIP is a globally significant virtual IP address, allocated by some control
   // or management plane component from a range defined for the ClusterSet.
   VIP string `json:"vip,omitempty"`
}

// ServiceStatus defines the observed state of a Service
type ServiceStatus struct {
   // BackendClusters defines the set of clusters providing the Service. It is updated
   // whenever a new axon:ServiceBinding is created and applied successfully.
   BackendClusters  []string `json:"backends",omitempty"`
}

// ObjectRef defines a reference to another k8s object - this is shown for completeness
// and we may be able to use the corev1.ObjectReference or similar built-in object instead.
type ObjectRef struct {
   Group     string `json:"group,omitempty"`
   Kind      string `json:"kind,omitempty"`
   Version   string `json:"version,omitempty"`
   Scope     string `json:"scope,omitempty"` // Cluster or Namespace scope for object
   Namespace string `json:"namespace,omitempty"`
   Name      string `json:"name"`
}

// ServiceBindingSpec defines the desired state of a ServiceBinding (i.e., a local service
// binding for a global service identifier)
type ServiceBindingSpec struct {
   // ServiceID defines the global service for which a local endpoint exists.
   ServiceID string   `json:"service,omitempty"`
   // ClusterID defines the cluster providing the endpoint (via the cluster's Gateways).
   ClusterID string   `json:"cluster,omitempty"`
   // ServiceRef references the local Kubernetes service object bound to the global service.
   ServiceRef ObjectRef `json:"serviceRef,omitempty"`
}

// ServiceImportSpec defines the desired state of a ServiceImport
type ServiceImportSpec struct {
   // ServiceID defines the global service that is being imported.
   ServiceID string `json:"service,omitempty"`
   // ClusterID defines the cluster making the Global service available for local consumption.
   ClusterID string `json:"cluster,omitempty"`
   // LocalName defines a local (DNS) name that can be used in the cluster to refer to the global
   // service. This should typically follow the cluster's Service naming convention
   // (e.g., <service>.<ns>.cluster.local).
   LocalName string `json:"localName,omitempty"`
}
```

The above CRDs are managed by a new Controller, running on the Broker
 cluster. Logically, the controller operates within a context of a ClusterSet (i.e., a single
 Broker namespace) and watches the new CRDs defined as well as existing Cluster objects. It
 reconciles desired and actual state based on the following logic:

1. `axon:Service` objects do not impact workload clusters directly. The controller merely
  sets a unique ID and VIP, if not already assigned. The `Status.Conditions` is set to
  "provisioned", with an empty array of `BackendClusters`.
1. `axon:ServiceBinding` objects trigger the creation of an `mcs:ServiceExport` in the
  relevant cluster. The controller confirms a valid specification (e.g., valid `ClusterID`,
  `ServiceID` and `ObjectRef` _syntax_ - note that we don't know at this point whether
  the `ObjectRef` refers to a valid Service object). The `mcs:ServiceExport` is created in
  accordance with the `ClusterID` and `ObjectRef`. We may wish to differentiate between a
  pending and validated binding (i.e., before the `ObjectRef` is determined valid and after).
  These may be captured in the `Status.Conditions` or by adding a `PendingClusters` array
  to the `Service.Status` structure. A workload cluster agent (either a modified Lighthouse
  component or an entirely new workload cluster controller), retrieves the `axon:ServiceBinding`
  object and creates the corresponding `mcs:ServiceExport` object in the correct namespace.
  The workload cluster agent can also update the binding status into the Broker cluster.
1. `axon:ServiceImport` object trigger the creation of an `mcs:ServiceImport` in the
  relevant cluster. Similar to `axon:ServiceExport` above, the controller confirms a valid
  specification before creating the `mcs:ServiceImport` specification for use by the target
  cluster. Similar `Status.Conditions` interaction may be used between the workload cluster
  agent and the Broker controller.

Optional implementation aspects and alternatives:

- The Broker controller may add a label based on the cluster identity (e.g., `ClusterID`
  or cluster name) to allow each cluster agent to efficiently filter for its own objects.
- `mcs:ServiceExport` and `mcs:ServiceImport` are references for objects in the same namespace
  and thus can not be used directly for independent naming. A workaround (barring changes to
  the MCS specification), is to replicate the equivalent `axon` objects to the workload clusters
  and create the MCS objects locally in each. A better (short term?) alternative would be to use
  the current Submariner workaround which uses predefined labels and annotations to communicate
  this information.
- Full reconciliation is required but not detailed above. For example, `ServiceBinding` status
  may change over time, as Cluster or Service objects might be deleted, etc.
- Since we don't propose to leverage any of the Lighthouse `ServiceExport` functionality,
  we could create a `GlobalIngressIP` object instead of creating `ServiceExport` objects. This
  requires decoupling GlobalNet behavior from `ServiceExport`s (which may already be
  sufficiently decoupled).
- Is a new workload cluster agent required or may we only tweak the behavior
  of an existing controller, such as Lighthouse?
- Currently (and in this proposal as well), workload cluster agents, such as Lighthouse, have
  access permissions on all objects in their Broker namespace. This allows them to read and,
  possibly, write objects belonging to other clusters. Running the agents in an administrator
  controlled namespace (e.g., "submariner-system"), provides a certain level of protection.
  However, in some cases a finer-grained access policy may be desirable. An alternative
  proposal is to create a `ClusterConfiguration` object, holding a map of all relevant
  configuration objects belonging to the cluster (e.g., a map of `ServiceExport` and
  `ServiceImport` split into `Spec` and `Status`). The workload agent is set up (on the
  Broker) with RBAC configured with `resourceNames` set to its specific cluster configuration
  object, allowing read access to the `Spec` and write access to the `Status` sub-resource.
  The Broker controller runs with a namespace wide role and is responsible for synchronizing
  all the individual object specification and status to the relevant ClusterConfiguration
  objects. We are not proposing to pursue this scheme in the first release and re-evaluate
  based on customer demand.

### Source Code Location

We propose a new top-level `submariner-io/axon-ccp` (short for for Axon Central Control plane).
 The repository will host additional components if needed in the future.

### Backward Compatibility

The changes are not backward-compatible (e.g., changes in components being run, RBAC
 permissions, etc.) and require that a ClusterSet is switched to use either the new
 or the existing service control planes. This could be done on a per ClusterSet basis or
 globally for all ClusterSets hosted on a Broker. For testing and evaluation, it may
 be worthwhile to allow dual control plane modes to run concurrently on the Broker, with
 each ClusterSet using only one of the mode at any given time. The selected mode can
 be set on the ClusterSet configuration object, defaulting to the current Submariner
 service control plane (e.g., a new `ServiceControlPlaneMode` string, with empty string
 indicating current implementation).

> Open: it is possible to emulate the current Submariner service behavior using
 the proposed control plane. This can be as simple as adding an `ImportToAll`
 flag to the `axon:Service` object or defining new `Export/ImportPolicy` objects.

#### Affected Components

- workload cluster
  - RBAC should allow only Submariner service accounts to create MCS objects. Leaving
    this open might interfere with expected processing. An option is to change
    relevant components to ignore local MCS objects that don't have a corresponding
    Broker cluster object.
  - Lighthouse
    - Avoid automatic syncing `ServiceExport`s to `ServiceImport`s on the Broker cluster.
    Possibly through runtime configuration or command line option to allow using the same
    code base.
    - Possible changes to name resolution to match specification in `axon:ServiceImport`
  - New service control plane component to interact with `axon:ServiceBinding` and
    `axon:ServiceImport` for `Status` updates (e.g., updated based on validity of `ObjectRef`).
- Broker Cluster
  - Run proposed Service control plane controller.
  - Changes in deployed components communicated to workload clusters based on service
    control plane mode.
  - Registration of Axon CRDs.
- More?

## Work Items

TBD, after review.
