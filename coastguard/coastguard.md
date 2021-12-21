# Submariner Multi-cluster network policy Support

## Summary
The purpose of this project is to introduce multi-cluster network policy to a submariner cluster.
Kubernetes Network Policy allows cluster administrators to configure granular isolation rules using Network Policy objects. A network policy is a specification of how groups of pods (using labels) are allowed to communicate with each other and other network endpoints.
The purpose of this project is to provide a mechanism for Network Policies across multiple clusters so that policy enforcement from a single cluster will seamlessly expand and work across clusters.
Currently, in a submariner environment which includes multiple k8n clusters, there is no way to define network policy between the two clusters. An administrator must manually collect ranges of ip addresses if he wants to limit traffic. Given the kubernetes pod network model, which is a flat network, this is not granular enough. 
Our service, named coastguard, will allow users to define multi-cluster network policies, and coastguard will translate the multi-cluster policies into relevant single-cluster network policies, and will keep those policies up to date after new deployments in each cluster. 

## Motivation
The submariner project’s goal is to provide connectivity and service discovery between 2 (or more) kubernetes clusters.  We will expand on common submariner use cases and describe the benefit from having network policy in each scenario.
1. Hybrid cloud:
Many times, the Private cloud runs workflows that are sensitive from a security standpoint, and as such we don’t want services on the public cloud to be able to access them. Being able to allow only some services to connect from the public cloud to the private is crucial.
2. Distributed Data:
In case of clusters in multiple regions, with replicas of the same service running in each cluster. The cloud-native Disaster Recovery will improve service availability while providing fault tolerance.
3. Disaster Recovery:
In case we have clusters deployed across active/passive sites, application is deployed in the passive site, but not receiving traffic, volume is replicated asynchronously. we need that when a disaster hits the active site, the application can be restored to the passive site.

For both scenarios B&C - having the ability to limit connectivity from the second site is needed, in case the second site is compromised. 

## requirements
1. Connected Clusters without any network policies should keep the default behavior
2. Allow frontend pods from a remote cluster in the same project to access the backend pods on the local cluster
3. Allow all the PODs from the same project in a remote cluster to access the backend POD on the local cluster
4. Allow frontend POD from a different namespace in a remote cluster to access the backend POD on the local cluster
5. Allow all the PODs from the same namespace which are either in the local cluster or remote cluster to access the logging POD on the local cluster
6. When a new cluster is added, , the resources on the new cluster need to be explored, and processed so it will comply with existing policies. 
When a cluster previously connected via submariner/federation is removed, any auxiliary resources, rules, or artifacts maintained by the submariner network policy implementation must be properly cleaned up.

### Goals

### Non-Goals

## Proposal Architecture

![image](https://user-images.githubusercontent.com/42589698/146993515-ba6bc995-8d3b-4642-b95f-dc56f4083502.png)

Coastguard aims to implement multi-cluster network policies by managing single-cluster network policies in each cluster. 
Our main challenge is tracking changes in deployments and pods in each cluster, and thus keeping the single-cluster network policies up to date. 
Coastguard is composed of:
  1.	Operator pattern = k8n controller + custom CRD. The custom crd will define the multi-cluster network policies.
  2.	Controller will use embedded etcd. as etcd is a distributed key-value store, etcd is responsible for updating the changes from the local cluster, to the controller in the remote cluster.
  3.	Controller tracks deployments using the Admiral API that federates resources to submariner data planes running in multiple clusters. Admiral api is an existing code component in submariner. 
Note: Coastguard is a repository in Submariner  
For security reasons, our technique uses the existing APIs and allows updates through them, so that an admin who controls the network policy of a cluster can only update changes in the CRD through the API and has no other authorization over the clusters. This model still has security implications that need to be explored and tested.
The core idea here is: that each controller has a reconciliation loop, where it re-generates a local policy based on remote network policy. Each controller has an embedded etcd server which holds the ip address and labels of all pods across all clusters. The controller will generate a “local”/regular network policy and will apply it to k8s api.
Creating the local network policy is done in an immutable way, and if in a given timeframe no changes occurred, the controller will simply apply the same local network policy to the k8s api server. 
As part of the project we will examine the sampling that the loop should withstand, it is necessary to find the threshold that will meet the required performance in the project, so we will examine how long it takes for a policy to sync between clusters and make sure that the admin will not expect to have connectivity earlier when a new policy will implement on the remote cluster. 
In our project we decided ignore:
  1.	Communication between the clusters not through the existing APIs, in order to make sure that the network policy was updated.
  2.	implementation for different CNIs and different network plugins (Calico, Cilium, etc.).
  3.	Authentication of network policies or remote pods based on tokens or certificates.

As open source projects, our idea should be accepted by the community, once it has passed the approval of the Red Hat team we will publish it to everyone, get reviews on the idea and may be required to update parts of it in favor of suitability for certain companies that use submariner.
The code will pass many tests and after the first pull request we will probably be required to update the code number of times, when we finish the process we will be able to be the maintainers of the code and continue to approve commits to it in the future. 


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
So, for each pod, we will have the following entries, where ‘$clustername’ for example is a placeholder for clustername.
1.	‘$clustername_$namespace_$podname’: ip address
2.	‘$clustername_$namespace_’: [list of labels]

We will need the following queries:
1.	get all pod ip, in cluster X with namespace Y
  query: etcdctl get --prefix ‘$clustername_$namespace`
2.	get all pod ips  in cluster X with namespace Y with label Z
  query:  etcdctl get --prefix ‘labels_$clustername_$namespace`

We will get from etcd a list of all pods, for every pod, all it’s labels. We will filter in memory for label Z.

3.	get all pod ips  in cluster with label Z
  query:  etcdctl get --prefix ‘labels_$clustername`
  Same idea as 2.

#note: ETCD is holding all the sensitive data in our network, the ETCD security is out of scoop in this project.
2.	CRD schema
our multi-cluster network policy CRD:

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
  - Ingress
  - Egress
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


### Overview of Proposed Changes

### Command Flow

## Alternatives

## Open Questions

