## Globalnet 2.0: Enhancements to Globalnet Implementation

## Summary

Currently, Submariner Globalnet implementation in [release 0.8](https://github.com/submariner-io/releases/releases/tag/v0.8.0)
annotates every Pod, Service and Node in the cluster with a globalIp to support Pod to remote
Service connectivity in the following manner.

Once a Pod is scheduled and enters Running state, it is annotated with a globalIp. To support
the connectivity, Globalnet programs the necessary egress rules on the active Gateway Node of
the cluster where the source IP of the Pod is modified to the corresponding globalIp. This
translation enables us to uniquely identify the traffic from the Pod, when it reaches the
destination cluster.

Similarly, a ClusterIP Service is annotated with a globalIp so that remote Pods can communicate
with it using its globalIp. Globalnet programs the necessary ingress rules on the active Gateway
node of the cluster, where the service resides. The ingress rules enable any incoming traffic
destined to the globalIp of the Service to be redirected to the corresponding ClusterIP Service.

In order to support connectivity from the Host Network to the remote services, Globalnet also
annotates every node in the cluster with a unique globalIp.

While this works fine for small clusters, where the number of Pods and Services are limited, for
large clusters it poses scalability and performance issues as we would eventually run out of
globalIps because of annotating every Pod/Service/Node object. Also, in this design, Pod to Pod
connectivity as well as Headless Services are not supported.

## Proposal

This enhancement proposes the following changes to make Globalnet implementation scale better.

Egress traffic:

1. By default, every cluster will be assigned two globalIps which will be used as egress IPs
   for any cross-cluster communication. We are allocating two globalIPs at the cluster level
   to avoid ephermeral port exhaustion issues.
2. Provision to allocate a single globalIp at the namespace/project level which takes precedence
   over the globalIp allocated to the cluster.
3. Provision to allocate a globalIp to a Pod (or a set of Pods) which has highest precedence
   over globalIPs at namespace level. If there are multiple policies, users can specify a priority
   field for each of the policy.

Ingress traffic:

4. Instead of annotating every ClusterIP Service in the cluster, only exported Services will be
   annotated with a globalIp.
5. For Headless Services, necessary ingress rules will be programmed to support connectivity to
   the backend Pods. In a Headless Service use-case, every backend Pod will be annotated with
   a unique globalIP as an Ingress IP. Clients can discover the associated globalIPs using Lighthouse
   and can connect to the globalIP assigned to the backend pods.
   For Pods backed by Headless Services, if the user does not explicitly request a globalIP using
   GlobalnetEgressIP with PodSelector that matches those Pods, the same globalIP assigned to the
   Pods will be used for both Ingress as-well-as Egress. However, if user creates a GlobalnetEgressIP
   with PodSelector that matches the backend pods of Headless Service, the globalIP assigned for
   the PodSelector takes precedence over individual Pod globalIPs as EgressIP.

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
avoid requesting explicit globalIP with podSelectors that match Headless Service backend Pods.

The proposal has its own set of Pros and Cons.

## Pros

Since we will be annotating only select K8s objects with globalIp, it will help in improved scalability.
Also, this translates to fewer iptable rules on the active Gateway node of the cluster which will
improve the overall performance of the solution.

In the current implementation, when a client Pod is scheduled, until it's annotated with a globalIp,
and the corresponding egress rules are programmed on the node, it was not able to communicate with
remote Services. This was creating some issues in-terms of user-experience. In this proposal, as we
have a globalIp at the cluster level, a Pod can communicate with remote Services right away which
will be closer to the Vanilla Submariner experience.

## Cons

A limitation with the proposed approach is that we will lose the unique identity for **every**
Pod which is supported today. For users that intend to use Network Policies across Clusters or
for debugging purposes, this can be mitigated by creating the necessary CRDs which create unique
globalIps at the Namespace/Pod level.

The following CRD is proposed to support the new design.

```Go
type GlobalnetEgressIP struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec               EgressIPSpec `json:"spec"`
}

type EgressIPSpec struct {
    // Selects the pods to which the GlobalnetEgressIP will be applied.
    // If multiple pods match the selector the same egressIP will be used for all those pods.
    // +optional
    Selector SelectorSpec `json:"selector,omitempty"`

    // Clients can request a specific globalIP to be allocated. If the requested
    // globalIP is part of globalCIDR of the Cluster and is not allocated to other
    // K8s objects, then it will be allocated, otherwise creation of the CRD will fail.
    // Valid values are empty string (""), or a valid IP address from the globalCIDR of
    // the Cluster. This field cannot be changed through updates.
    GlobalIP  string      `json:"global_ip"`
}

type SelectorSpec struct {
    // Selects the pods to which this GlobalnetEgressIP object applies. The allocated
	// global_ip will be used for all the Pods selected by this field.
    PodSelector metav1.LabelSelector `json:"podSelector"`

    // Associated priority to be used for the selected pods.
    // Its value should be between 1 (lowest) and 1000 (highest) inclusive.
    // When this value is omitted, the default value of 1 is used.
    // +optional
    Priority    int32 `json:"priority,omitempty"`
}
```

### User Stories

1.User has two (or more) clusters with Overlapping CIDRs and requires the Pods in one
  Cluster be able to discover and talk to exported Services.

  To achieve this use-case, user can simply deploy Submariner Globalnet on all the participating
  clusters. By default, a globalIP will be assigned at the Cluster level which will be used as
  egressIP for inter-cluster communication and necessary Services can be exported.

2.In a Globalnet deployment user wants the Pods in a particular namespace to use a unique globalIP
  instead of the globalIP assigned at the cluster level.

  To achieve this, one can apply the following CRD.

```yaml
   apiVersion: submariner.io/v1alpha1
   kind: GlobalnetEgressIP
   metadata:
     name: ns-egressip
     namespace: myproject
   spec:
     globalIP: (optional)
```

3.In a Globalnet deployment, user wants a selected Pod (or set of Pods) in a particular namespace
  to use a unique globalIP as egressIP for cross-cluster communication.

  To achieve this, one can apply the following CRD.

```yaml
   apiVersion: submariner.io/v1alpha1
   kind: GlobalnetEgressIP
   metadata:
     name: db-pods
     namespace: myproject
   spec:
     globalIP: (optional)
     selector:
       podSelector:
         matchLabels:
           role: db
       priority: 100 (optional) 
```

4.In a Globalnet deployment, user wants a selected Pod (or set of Pods) in a particular namespace
  to use a unique globalIP both as ingress and egressIP for cross-cluster communication.

  To achieve this, user can create a Headless Service (backed by a Deployment or StatefulSets) and
  Globalnet will ensure that all the backend Pods that belong to the Headless Service are assigned
  a globalIP. User is also expected **not** to create any GlobalnetEgressIP object which selects the
  backendPods that match the Headless Service.

![GlobalnetTopology](./images/globalnet-headless-svc.png)

  In the above example, each of the http pods will get its own globalIP which can be used for
  both ingress and egress communication.

  However, if the user wants egress traffic from the http Pods to carry a unique globalIP,
  its possible by explicitly requesting a GlobalnetEgressIP with PodSelectors matching Headless
  Service Pods as shown below.

```yaml
   apiVersion: submariner.io/v1alpha1
   kind: GlobalnetEgressIP
   metadata:
     name: http-pods
     namespace: myproject
   spec:
     globalIP: (optional)
     selector:
       podSelector:
         matchLabels:
           app: http
   status:
     globalIP: 169.254.0.100
```

![GlobalnetTopology](./images/globalnet-svc.png)

## Alternatives

It is possible to support Pod to Pod communication even with Globalnet. However, to support this, it
requires programming additional IPtable rules on the nodes which will have an impact on the performance
of the solution. This proposal provides some limited Pod to Pod connectivity via the Headless Services.
If user is interested in Pod to Pod communication for all the Pods, it is recommended to install
Clusters with non-overlapping CIDRs and use Vanilla Submariner instead of Globalnet.

## Work items

1. Support annotation at the cluster level and program necessary egress rules.
2. Support annotation at namespace level which takes precedence over globalIp at cluster level.
3. Support annotation to Pods (or set of Pods) which has the highest precedence while
   honouring the priority.
4. Annotate only exported Services instead of annotating all Services.
5. Support Headless Services and program ingress/egress rules for the backend Pods.
6. Avoid dependency on the iptable chains programmed by IPtables kube-proxy driver.
7. Implement necessary unit tests for each of the use-cases.
8. Implement necessary e2e tests to validate various use-cases.
9. Support upgrade from 0.8 release to the newer implementation.
