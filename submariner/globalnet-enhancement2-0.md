## Globalnet 2.0: Enhancements to Globalnet Implementation

## Summary

Currently, Submariner Globalnet implementation in [release 0.9](https://github.com/submariner-io/releases/releases/tag/v0.9.0)
annotates every Pod, exported Services and Nodes in the cluster with a globalIP to support Pod to remote
Service connectivity in the following manner.

Once a Pod is scheduled and enters Running state, it is annotated with a globalIP. To support
the connectivity, Globalnet programs the necessary egress rules on the active Gateway Node of
the cluster where the source IP of the Pod is modified to the corresponding globalIP. This
translation enables us to uniquely identify the traffic from the Pod, when it reaches the
destination cluster.

Similarly, a ClusterIP Service is annotated with a globalIP so that remote Pods can communicate
with it using its globalIP. Globalnet programs the necessary ingress rules on the active Gateway
node of the cluster, where the service resides. The ingress rules enable any incoming traffic
destined to the globalIP of the Service to be redirected to the corresponding ClusterIP Service.

In order to support connectivity from the Host Network to the remote services, Globalnet also
annotates every node in the cluster with a unique globalIP and this functionality will be preserved
even in the new implementation.

While this works fine for small clusters, where the number of Pods and Services are limited, for
large clusters it poses scalability and performance issues as we would eventually run out of
globalIPs. Also, in this design, Pod to Pod connectivity as well as Headless Services are not supported.

## Proposal

This enhancement proposes the following changes to make Globalnet implementation scale better.

Egress traffic:

* By default, every cluster will be assigned a configurable number of globalIPs which will be
  used as egress IPs for any cross-cluster communication. Support for more than a single globalIP
  at the cluster level is done to avoid ephemeral port exhaustion issues.
* Provision to allocate a set of globalIPs at the namespace/project level which takes precedence
  over the globalIPs allocated to the cluster. All Pods in the namespace will use the same globalIPs
  as EgressIPs.
* Provision to allocate globalIPs to a set of Pods in a namespace.
  Such Pods will use globalIPs that take precedence over any globalIPs allocated at the default
  namespace level.
* Applications on Host Network trying to access remote services will be using the ClusterGlobalEgressIP.

Ingress traffic:

* Instead of assigning a globalIP every ClusterIP Service in the cluster, only exported Services
  will be assigned a globalIP. Globalnet internally creates a GlobalIngressIP CR for every
  exported service. Users will be able to query GlobalIngressIP but will not be able to create it.
* For Headless Services, necessary ingress rules will be programmed to support connectivity to
  the backend Pods. In a Headless Service use-case, every backend Pod will be assigned a unique
  globalIP as an Ingress IP. Clients can discover the associated globalIP using Lighthouse and
  can connect to the globalIP assigned to the backend pods. For Pods backed by Headless Services,
  if the user does not explicitly request a globalIP via GlobalEgressIP CRD, the same globalIP
  assigned to the Pods will be used for both Ingress as-well-as Egress. However, if user creates
  a GlobalEgressIP that matches the namespace, or the backend pods of Headless Service, the globalIP
  assigned via GlobalEgressIP takes precedence over individual Pod globalIPs as EgressIP.

Both EgressIP as well as IngressIPs will be allocated from the same globalCIDR Pool used in
the Cluster. Globalnet will not support an EgressIP/IngressIP that is outside of globalCIDR
Pool.

![GlobalnetTopology](./images/globalnet-topology.png)

If the external application in a remote Cluster is interested in consuming the exported
Service, it will have to reach the globalIP assigned to the exportedService and the response
from the backend Pods will have the source as the globalIP of the exportedService.

On the other hand, when the backend Pod tries to talk to the external application in remote
Cluster, the external application will see that traffic is coming with a globalIP that is
assigned either at the Cluster level, or namespace level, or the one requested by the user
at Pod level. This asynchronous behavior will be analogous to the behavior seen with standard
K8s applications when they talk to any Services. Applications desiring to have the same IPaddress
for both ingress and egress traffic can create Headless Services in front of them and should
avoid requesting explicit globalIP that match Headless Service backend Pods.

The proposal has its own set of Pros and Cons.

### Pros

Since we will be allocating globalIPs to only select K8s objects, it will help in improved scalability.
Also, this translates to fewer iptable rules on the active Gateway node of the cluster which will
improve the overall performance of the solution.

### Cons

A limitation with the proposed approach is that we will lose the unique identity for **every**
Pod which is supported today. For users that intend to use Network Policies across Clusters or
for debugging purposes, this can be mitigated by creating the necessary GlobalEgressIP CRDs
which create unique globalIPs at the Namespace/Pod level.

### Caveats

In the current implementation, when a client Pod is scheduled, until it's annotated with a globalIP,
and the corresponding egress rules are programmed on the node, it was not able to communicate with
remote Services. This was creating some issues in-terms of user-experience. In this proposal, as we
have a globalIP at the cluster level, a Pod can communicate with remote Services right away which
will be closer to the Vanilla Submariner experience.

## Design Details

The following CRD is proposed to support the new design.

```Go
type ClusterGlobalEgressIP struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // Spec is the specification of desired behavior of ClusterGlobalEgressIP object.
    Spec               ClusterGlobalEgressIPSpec `json:"spec"`

    // Observed status of ClusterGlobalEgressIP. Its a read-only field.
    // +optional
    Status             EgressIPStatus `json:"status,omitempty"`
}

type ClusterGlobalEgressIPSpec struct {
    // The number of globalIP's requested at the cluster-level.
    // Globalnet Controller will allocate the requested number of contiguous GlobalIPs for
    // this ClusterGlobalEgressIP object.
    // If unspecified, NumIPs defaults to 1 and max allowed is restricted to 10.
    // +optional
    NumIPs  *int     'json:"numIPs",omitempty"`
}

type GlobalEgressIP struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // Spec is the specification of desired behavior of GlobalEgressIP object.
    Spec               GlobalEgressIPSpec `json:"spec"`

    // Observed status of GlobalEgressIP. Its a read-only field.
    // +optional
    Status             EgressIPStatus `json:"status,omitempty"`
}

type GlobalEgressIPSpec struct {
    // The number of globalIP's requested at the namespace level or for selected pods in a namespace.
    // Globalnet Controller will allocate the requested number of contiguous GlobalIPs for this
    // GlobalEgressIP object.
    // If unspecified, NumIPs defaults to 1 and max allowed is restricted to 10.
    // +optional
    NumIPs  *int     'json:"numIPs",omitempty"`

    // Selects the pods whose label match the definition. This is an optional field and
    // in case it is not set, it results in all the Pods selected from the namespace in which the
    // CR is created.
    // +optional
    PodSelector metav1.LabelSelector `json:"podSelector,omitempty"`
}

const (
    // GlobalEgressIPSuccess means that Globalnet was able to successfully allocate the GlobalIP.
    GlobalEgressIPSuccess GlobalEgressIPStatus = "Success"

    // GlobalEgressIPError means that Globalnet was unable to allocate the GlobalIP.
    GlobalEgressIPError GlobalEgressIPStatus = "Error"
)

type EgressIPStatus struct {
    // Status is one of {"Success", "Error"}
    Status GlobalEgressIPStatus `json:"status,omitempty"`

    // +optional
    Message *string `json:"message,omitempty"`

    // The list of GlobalIPs assigned via this GlobalnetEgressIP object.
    GlobalIPs []string `json:"globalIPs"`
}
```

A GlobalIngressIP object will be created for every exported Service in the namespace where the
service is exported. For Headless Services, backed by Deployments or StatefulSets, Globalnet creates
a GlobalIngressIP object for every backend pod. It uses "svc-" and "pod-" as prefixes while naming
these objects and the globalIP assigned to the respective object can be queried from the Status field.

```Go
type GlobalIngressIP struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // Spec is the specification of desired behavior of GlobalIngressIP object.
    Spec GlobalIngressIPSpec `json:"spec"`

    // Observed status of GlobalIngressIP. Its a read-only field.
    // +optional
    Status GlobalIngressIPStatus `json:"status,omitempty"`
}

type GlobalIngressIPSpec struct {
    // TargetType can be ClusterIPService or a Pod
    Target TargetType `json:"target"`

    // If the TargetType points to a Pod that belongs to a Headless Service, it uses the
    // name of the corresponding Service.
    // +Optional
    ServiceName string `json:"serviceName,omitempty"`

    // Since the Name of GlobalIngressIP will be prepended with "svc-" or "pod-" based on
    // the object, we may have to truncate the name in some cases when the length exceeds the
    // maximum length that is allowed by K8s. In such cases, the objRef will include the
    // full name of the object.
    objRef *LocalObjectReference `json:"objRef,omitempty"`
}

type GlobalIngressIPStatus struct {
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type"`

    // The GlobalIP assigned to the respective object.
    // +optional
    GlobalIP string `json:"globalIP"`
}

type TargetType string

const (
    ClusterIPService   TargetType = "ClusterIPService"
    HeadlessServicePod TargetType = "HeadlessServicePod"
)
```

When Submariner Globalnet is deployed in a Cluster, it auto-allocates the configured number of
`globalIPs` and programs the necessary egress rules on the active Gateway node of the Cluster.
This is done internally by creating `ClusterGlobalEgressIP` CR object with the name `cluster-default`.
Users can query the CR to check the allocated GlobalIPs at the Cluster level. If admin creates
additional ClusterGlobalEgressIP CRs that match the Cluster scope, the status field of those CRs will
be marked as error with appropriate message. Basically only `cluster-default` object will be supported.

Only an admin user can create the `ClusterGlobalEgressIP` object and regular users will only be able
to create `GlobalEgressIP` objects.

### Backward Compatibility

In order to support the newer implementation, a combination of ipSets with multiple IPTable chains
will be used. Also, we will internally use configMaps to maintain the state of allocated GlobalIPs
and get rid of annotations on the objects. Because of this, backward compatibility cannot be maintained
with the existing implementation of Globalnet.

### User Stories

#### Default scenario

User has two (or more) clusters with Overlapping CIDRs and requires the Pods in one Cluster be able
to discover and talk to exported Services. To achieve this use-case, user can simply deploy Submariner
Globalnet on all the participating clusters. By default, a globalIP (or a set of globalIPs) will be
assigned at the Cluster level which will be used as egressIPs for inter-cluster communication and
necessary Services can be exported.

#### Unique globalIP for a namespace

In a Globalnet deployment user wants all the Pods in a namespace `ns1` to use a unique globalIP
instead of the globalIP assigned at the cluster level. To achieve this, one can apply the following
CRD in the namespace `ns1`.

```yaml
   apiVersion: submariner.io/v1alpha1
   kind: GlobalEgressIP
   metadata:
     name: ns-egressip
     namespace: ns1
   spec:
     numIPs: 1
```

#### Unique globalIP for selected Pods in a namespace

In a Globalnet deployment, user wants a selected Pod (or set of Pods) in a particular namespace to use a
unique globalIPs as egressIP for cross-cluster communication. To achieve this, one can apply the following
CRD in the namespace `ns1`.

```yaml
   apiVersion: submariner.io/v1alpha1
   kind: GlobalEgressIP
   metadata:
     name: db-pods
     namespace: ns1
   spec:
     podSelector:
         matchLabels:
           role: db
     NumIPs: 2
```

#### GlobalIP for exported ClusterIP service

In a Globalnet deployment, user wants a specific service to be made available to remote clusters.
For this, user can create a ClusterIP Service and export the Service. Globalnet will allocate a
globalIP for ingress connectivity such that any external applications can connect to this Service.

```shell
  kubectl -n default create deployment nginx --image=nginxinc/nginx-unprivileged:stable-alpine
  kubectl -n default expose deployment nginx --port=8080
  subctl export service --namespace default nginx
```

#### GlobalIP for exported Headless Service

In a Globalnet deployment, user wants a selected Pod (or set of Pods) in a particular namespace
to use a unique globalIP both as ingress and egressIP for cross-cluster communication.

To achieve this, user can create a Headless Service (backed by a Deployment or StatefulSets) and
Globalnet will ensure that all the backend Pods that belong to the Headless Service are assigned
a globalIP. User is also expected **not** to create any GlobalEgressIP object which selects
the backendPods that match the Headless Service.

![HeadlessService](./images/globalnet-headless-svc.png)

In the above example, each of the http pods will get its own globalIP which can be used for
both ingress and egress communication.

However, if the user wants egress traffic from the http Pods to carry a unique globalIP,
its possible by explicitly requesting a GlobalEgressIP with PodSelectors matching
Headless Service Pods as shown below.

```yaml
   apiVersion: submariner.io/v1alpha1
   kind: GlobalEgressIP
   metadata:
     name: http-pods
     namespace: myproject
   spec:
     podSelector:
       matchLabels:
         app: http
```

![Headless Service with EgressIP](./images/globalnet-svc.png)

## Alternatives

This proposal does not support direct pod to pod connectivity, but it supports Headless Services.
When an Headless Service is exported, each backend pod of a Headless Service will get a unique globalIP
that can be queried via Lighthouse DNS and client Pods can talk to the respective backend pods.

If a user is interested in Pod to Pod communication for all the Pods, it is recommended to install
Clusters with non-overlapping CIDRs and use Vanilla Submariner instead of Globalnet.

## Work items

1. Support creation of ClusterGlobalEgressIP and GlobalEgressIP CRDs and the associated Cluster Roles.
   * [Create Clientset, informers and listers for GlobalnetEgressIP CRD](https://github.com/submariner-io/submariner/issues/1157)
   * [Create ClusterRoles for accessing GlobalnetEgressIP CRD](https://github.com/submariner-io/submariner-operator/issues/1114)
2. Provide configuration parameter either via ConfigMap or Env variable to specify
   the number of default GlobalIPs that have to be allocated at cluster level.
   * [Configuration parameter for default number of globalIPs at Cluster level](https://github.com/submariner-io/submariner-operator/issues/1116)
3. [Support Globalnet EgressIPs at the cluster level and program necessary egress rules.](https://github.com/submariner-io/submariner/issues/1163)
4. [Support Globalnet EgressIPs at namespace level which takes precedence over
   globalIPs at cluster level.](https://github.com/submariner-io/submariner/issues/1164)
5. [Support Globalnet EgressIPs to Pods (or set of Pods) which has the highest precedence.](https://github.com/submariner-io/submariner/issues/1165)
6. [Allocate globalIP only to exported Services.](https://github.com/submariner-io/submariner/issues/720)
7. [Support Headless Services and program ingress/egress rules for the backend Pods.](https://github.com/submariner-io/submariner/issues/732)
8. This is a good to have internal enhancement and does not modify the user experience.
   [Avoid dependency on the IPtable chains programmed by IPtables kube-proxy driver.](https://github.com/submariner-io/submariner/issues/1166)
9. Implement necessary unit tests for each of the use-cases.
10. [Implement necessary e2e tests to validate various use-cases.](https://github.com/submariner-io/submariner/issues/1167)
11. [Update documentation.](https://github.com/submariner-io/submariner-website/issues/455)
12. [Document how to upgrade from 0.8 release to the newer implementation.](https://github.com/submariner-io/submariner-website/issues/456)
