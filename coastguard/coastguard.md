# Submariner Multi-cluster network policy Support

## Summary
The purpose of this project is to introduce multi-cluster network policy to a submariner cluster.

Kubernetes Network Policy allows cluster administrators to configure granular isolation rules using Network Policy objects. A network policy is a specification of how groups of pods (using labels) are allowed to communicate with each other and other network endpoints.

The purpose of this project is to provide a mechanism for Network Policies across a cluster-set such that policy enforcement from a single cluster will seamlessly expand and work across clusters.

Currently, in a submariner environment which includes multiple k8s clusters, there is no way to define network policy between the two clusters. An administrator must manually collect ranges of ip addresses if he wants to limit traffic. Given the kubernetes pod network model, which is a flat network, this is not granular enough. 

The proposed service, named Coastguard, will allow cluster administrators to define multi-cluster network policies, and coastguard will translate the multi-cluster policies into relevant single-cluster network policies, and will keep those policies up to date after new deployments in each cluster. 

## Motivation
With the advent of [multi cluster services api](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api) the need for restricting traffic across a cluster-set has become more apparent.

When a service is imported into a cluster, the imported service and it's underlying pods are accessible from all pods in the importing cluster. 

Limiting traffic to the service and it's underlying pods by other pods can't be done by semantic rules (namespaces, labels), which greatly limits the options for a cluster administrator wanting to limit and secure his cross-service traffic.
## Requirements
1. Connected Clusters without any network policies should keep the default behavior.
2. Allow limiting traffic to/and from a service based on namespace.
3. Allow limiting traffic to/and from the underlying pods of a service based on labels and/or namespace.
4. When a cluster previously connected via submariner is removed, any auxiliary resources, rules, or artifacts maintained by the submariner network policy implementation must be properly cleaned up.

## Proposal Architecture

![image](https://user-images.githubusercontent.com/42589698/146993515-ba6bc995-8d3b-4642-b95f-dc56f4083502.png)

Coastguard aims to implement multi-cluster network policies by managing single-cluster network policies in each cluster. 
Our main challenge is tracking changes in deployments and pods in each cluster, and thus keeping the single-cluster network policies up to date. 
Coastguard is composed of:
  1.	Operator pattern = k8s controller + custom CRD. The custom crd will define the multi-cluster network policies.
  2.	Controller will use embedded etcd. as etcd is a distributed key-value store, etcd is responsible for updating the changes from the local cluster, to the controller in the remote cluster.

  3.	Controller tracks deployments using the Admiral API that federates resources to submariner data planes running in multiple clusters. Admiral api is an existing code component in submariner. 

For security reasons, our technique uses the existing APIs and allows updates through them, so that an admin who controls the network policy of a cluster can only update changes in the CRD through the API and has no other authorization over the clusters. This model still has security implications that need to be explored and tested.

The core idea here is: that each controller has a reconciliation loop, where it re-generates a local policy based on remote network policy. Each controller has an embedded etcd server which holds the ip address and labels of all pods across all clusters. The controller will generate a “local”/regular network policy and will apply it to k8s api.

Creating the local network policy is done in an immutable way, and if in a given timeframe no changes occurred, the controller will simply apply the same local network policy to the k8s api server. 

As part of the project there is a need to examine the interval for which the reconciliation loop should run. It is necessary to find the threshold that will meet the required performance in the project, as this interval represents the biggest trade off in this proposal.


In our proposal we currently ignore:
1. implementation for different CNIs and different network plugins (Calico, Cilium, etc.).
2. Authentication of network policies or remote pods based on tokens or certificates.
 


## Design Details
The following is a diagram of the main “reconciliation loop”. This is our main flow, and any subsequent will use this flow to do the bulk of the work.

![image](https://user-images.githubusercontent.com/42589698/146993706-c19bdfa6-be3a-486e-8d46-fb875f169d78.png)

We will have 4 main flows:
  1.	a clusters joins the cluster set 
  2.	a deployment change happens on any cluster  
  3.	a user adds/changes new multi-network policy is added
  4.	a cluster is removed from the set 

Flows 2-3 have no special additions beyond executing the reconciliation loop.
For flow 1 - when a cluster first joins the cluster set, and the coastguard controller runs for the first time, the controller will need to map all existing namespaces and pods on the k8s cluster. It will do so by running the API equivalent of `kubectl get namespaces` and `kubectl get pods`.

For flow 4 - once a cluster leaves the cluster set, the coastguard controller on the leaving cluster should delete all it’s entries from etcd. 

## Component/Service Coast guard controller

1. ETCD structure
In general, etcd is a key value store. It has the ability to query based on key prefixes.
So, for each pod, we will have the following entries, where `$clustername` for example is a placeholder for clustername.
1.	`$clustername_$namespace_$podname`: ip address
2.	`label_$label_$clustername_$namespace_$podname`: ip address 

We will need the following queries:
1.	get all pod ip, in cluster X with namespace Y
  query: etcdctl get --prefix `$clustername_$namespace`
2.	get all pod ips  in cluster X with namespace Y with label Z
  query:  etcdctl get --prefix `label_$labelZ_$clusternameX_$namespaceY`

3.	get all pod ips  in cluster with label Z
  query:  etcdctl get --prefix `label_$labelZ_$clusternameX`
  Same idea as 2.

#note: ETCD is holding all the sensitive data in our network, the ETCD security is out of scoop in this project.
2.	CRD schema
our multi-cluster network policy CRD:
```
apiVersion: networking.submariner.io/v1
kind: MultiClusterNetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  clusterSelector: cluster1
  podSelector:
    matchLabels:
      role: db

  policyTypes:
    ingress:
    - from:
      - ipBlock:
          cidr: 172.17.0.0/16
          except:
          - 172.17.1.0/24
      - namespaceSelector:
          matchLabels:
            key: value
      - podSelector:
          matchLabels:
            key: value
      - clusterSelector: value
        
      ports:
      - protocol: TCP
        port: 6379
    egress:
    - to:
      - ipBlock:
          cidr: 172.17.1.0/16
          except:
          - 172.17.2.0/24
      - namespaceSelector:
          matchLabels:
            key: value
      - podSelector:
          matchLabels:
            key: value
      - clusterSelector: value
```

## Open Questions

1. should we forgo the use of clusterSelector and use just namespaces based on the principal of [namespace sameness](https://github.com/kubernetes/community/blob/master/sig-multicluster/namespace-sameness-position-statement.md)
2. Can we integrate `EndpointSlice` in fetching pod ip addresses?
