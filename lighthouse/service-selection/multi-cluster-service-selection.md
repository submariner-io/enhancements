# Multi-cluster service selection optimization - EP

## Summary

This EP, propose a 2 stage development process for improving the
load balancing of service-to-service communication
in the multi-cluster setup of Kubernetes using
Submariner.

## Motivation

While the world is moving towards micro-services application-based architecture.
More organizations combine multi-cluster and multi-cloud providers,
for reasons such as
High-Availability, disaster recovery (redundancy),
scalability, performance, multi-tenant isolation, and others.
The above introduces traffic pricing changes and
performance challenges that need to be addressed.

### Goals

- Describe the short theoretical solution for optimizing
  the total `cost` of Kubernetes service-to-service
  traffic, given a certain multi-cluster deployment.
- Define minimal API for Submariner to enable
  control of such optimization
- Specify the basic modules needed to accomplish and implement the solution
- Allow 2 stages rollout of gradual improvement
  and model implementation

    1. Weighted-Round-Robin (3 months) - improve
       current Round-Robin implementation
    1. Optimize-Load-Balancing (6 months) - deploy
       full solution

- Leave room for future extension and new use cases

### Non-Goals

- Define specific implementation details beyond general API behavior.
- Optimizing deployment cost itself
  (i.e reducing the cost of paying for compute power)
  under the same assumptions is out of this EP scope).

## Proposal

### User Stories

#### Operator can configure load balancing techniques

1. Round-Robin:

The distribution of traffic (i.e request) among Service-Imports of
the same service type will be equal in expectation.

1. Simple Weighted-Round-Robin:

Each Service-Import will be assigned with a weight according to predefined
parameters (e.g latency, RTT, pricing, etc), the `lower`
the weight the `higher` the probability to `select` that
specific service, in such way in expectation the traffic
will distribute according to the weight.
For example we have a system with 3 clusters (c1,c2,c3)
and 2 exports for `service-1` from 2 clusters (c2,c3) in
each cluster we will assign an appropriate weight for each
export. An option will be `c1 {c2: 10, c3: 5}` (note that
c1 does not have a local version of `service-1`),
`c2 {c2: 0, c3: 4}`, `c3 {c2: 4, c3: 0}`.
Now, let's take cluster `c1` for example that needs to
distribute the load between `c2` and `c3` for `service-1`.
in expectation over time a `1/3` (33%) of the traffic will
be passed to `c3` and `2/3` (67%) will be passed to `c2` -
inversed of their weight.
Note about `zero weight` (`weight=0`). We have already
implemented a mechanism that prefers local cluster service
over remote ones, using `lower` weight -> `higher`
the probability we will make a safety mechanism to return the
cluster with the `weight=0` always probability will equal = 1.
For safety, we will not allow more than 1 cluster to have `weight=0`.

1. Optimize:

The system will calculate a smart weight for Service-Imports of
the same kind in a way that will minimize some `cost` (will be
explain shortly) the operator defines.
From there the traffic (i.e request) distribution will work in
like the Weighted-Round-Robin fashion.

### Requirements

1. The solution must be resilient and distributed

    1. Each cluster should make an independent service selection/routing decision
    1. Adaptive - make the system as adaptive as possible for changes
       such as demand, outage, bugs, etc

1. Topology - optimizing only next-hop -
  i.e without any relation to the full application graph,
  simple yet naive approach.

### Caveats

1. Some functionality might need to break the
   Kubernetes Multi-cluster API design by the
   community. As this project also have academic value, in
   such event we shall fork the solution and prove the worth
   of breaking the community accepted API.

### Risks and Mitigations

As our solution is distributed we anticipate several
classic distributed load balancing problems:

1. Herd behavior - where services form different (or even
   same) clusters will send a high volume of their traffic to
   the same destination
1. Starvation - where servers are idle and get no traffic
   sent to them.
1. High volume of data sharing

We plan to mitigate these risks by splitting the
the solution into Control Plan and Data Plan
distributed solution.
While Kubernetes clusters will share data through
the Broker component which will save the two ways
data sharing, as well where we will have a central
point of view of all the services of the same type
where we can optimize in central fashion to
prevent herd behavior.
Besides, we plan to fine-tune and find the best
interval for the data to be shared and the
optimization problem to be solved, to minimize data
sharing and system overloading.
The starvation problem we are going to solve as part of the
model which will be described shortly.

## Design details

![Optimization-Model](ServiceSelection.gif)

### Definitions

#### Cost

A generic cost function that will help us calculate the weights

1. Parameters:

    1. Latency RTT - infrastructure level
    1. Networking Egress cost

        1. Internet - when leaving the cloud provider.
        1. Cross-Region - when staying in the same
        cloud provider,
        but transferring data between regions.
        1. Cross Zone - staying within the same region, but
        transferring data between zones.

1. Requirements - the operator will be able to
 mark the trade-offs between cost parameters,
 such as sacrificing latency for better
 pricing, best performance, etc
1. Type of functions to be explored

    1. [weighted sum](https://en.wikipedia.org/wiki/Weighted_sum_model)

#### Theoretical Models

Two models, which will help us to gradually roll the optimization

1. Weighted-Round-Robin:
   We have `C` be a set of clusters.
   We have a request `r` entering our system (in our
   Lighthouse layout
   means a DNS request that needs to be translated into an
   IP address of the target service),
   a request `r` has a `source` Cluster within `C`,
   a set of destinations clusters `destinations`
   that supports that request, i.e the clusters that
   have the service that the request is targeted to,
   and a type `t` representing the service type that
   handle this request.
   We define a cost function between two clusters,

   ```python
   func cost(cluster1, cluster2) -> float64
   ```

   which indicates the penalty we would pay to send a
   request from `cluster1` to `cluster2`.
   From the cost, we deduce the weights in inverse
   proportions, such that the higher the cost the
   lower the weight

   ```python
   weight(cluster1, cluster2) = 1 / cost(cluster1, cluster2))
   ```

   This is the weight for the simple Weighted-Round-Robin
   algorithm which will eventually distribute the load

1. Optimized Model:
   We have the same set of clusters and requests as before.
   in addition to several optimization parameters.
   We define

   ```python
   func cap(cluster1) -> float64
   ```

   which defines the capacity `cluster1` have for handling
   requests of the same type of `r` (measured in RPS).
   We define `N(s)` as an estimated (to be
   discussed) number of requests
   (i.e load - also measured in RPS) of a certain we
   would like to send from the `source` cluster at a
   certain time.
   We finally define our optimization variable
   `n(s,t)` as the indicator variable representing
   the number of requests `source` cluster should
   generate on a `target` cluster.
   With that information in mind we would like to minimize:

   ```python
   sum(cost(src_c, dst_c) * n(src_c, dst_c) for src_c in C for dst_c in C)
   ```

   But we have some constraints we need to comply with:

    1. Request cannot be dropped:

     ```python
     for src_c in C:
       sum(n(src_c, dst_c) for dst_c in C) = N(src_c)
     ```

    1. Don't suppress targets capacity:

     ```python
     for dst_c in C:
       sum(n(src_c, dst_c) for src_c in C) <= cap(dst_c)
    ```

    1. Liveness:

    ```python
    for dst_c in C:
      for src_c in C:
        n(src_c, dst_c) >= 1
    ```

    After solving the the optimization problem each cluster will be
    assign the weight: `w(s,t) = n(s,t) \ N(s)`

### Basic API and features

Allow the operator a simple way to indicate
the desire load balancer for service discovery,
with optimal parameters through the `Submariner CRD`.

1. Option 1 - Add the following properties:
    1. `serviceDiscoveryLoadBalancing: [round-robin, weighted-round-robin, optimized]`
    1. `optimizedLoadBalancingPrams: [cost, performance, mixed]`

`cost` - will optimized communication cost
`performance` - will optimized performance (latency and RTT wise)
`mixed` - will try to reach a balance between minimal cost and best performance

`serviceDiscoveryLoadBalancing` is relevant only for `serviceDiscoveryEnabled: true`
`optimizedLoadBalancingPrams`
is relevant only for `serviceDiscoveryLoadBalancing: optimized`

### Implementation and modules

All load balancing techniques are working in stochastic
coordination and using probability to avoid herd behavior
and server starvation, some are more prone to errors than
others and their performance and resilience shall be
tested according to the testing plan provided in the
following section.

#### Weighted-Round-Robin

The first implementation stage will be implemented
the [Weighted-Round-Robin](https://en.wikipedia.org/wiki/Weighted_round_robin) load
balancing mechanism in the Lighthouse Plugin.

1. Load-Balancing module - a small lightweight Go library
   that will implement and abstract the two load balancing
   techniques (Round-Robin, Weighted-Round-Robin, etc) away
   from the main implementation, it shall be imported and used
   within the Lighthouse Plugin.
1. Metrics module - an agent on each cluster (still need to
   decide/understand if it is already implemented in
   Submariner Gateway, or if we wish to implement it someplace
   else). The metrics module will collect and share the
   relevant data for the cost parameters
    1. Latency - will be measure using a
    [HTTPing](https://github.com/flok99/httping) tool
    1. Pricing - will be extracted from cloud provider pricing API
    [1](https://calculator.aws/#/createCalculator)
    [1](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/using-ppslong.html)
1. Optimization module - the optimization module will be
   part of the Broker cluster control plan, it will gather the
   metrics from the different metrics modules calculate the
   weights for each service and update them in the different clusters.

- If time will pressure, or we find it better we might
  combine the Metrics and Optimization modules and
incorporate them withing the Lighthouse for local simple
and easy solution.

#### Optimized

1. Load-Balancing module - repurpose the module from the first stage
1. Metrics module - repurpose the module from the first stage
1. Optimization module - similarly to the first stage, the module
   will be part of the
   Broker, just now it will solve the optimization problem described
1. Estimator Module - will be in-charge of estimating capacity and load if needed

#### Submariner modules responsibilities

- Broker

    1. Extend the Broker API to enable clusters to
      provide details about latency and load
    1. Implement a dedicated controller to build and solve the optimization problem
    1. Publish back to the clusters the appropriate weights

- Clusters

    1. Data collection and publishing:

        1. Use Kubernetes/Submariner built-in metrics
          controller to publish latency to the Broker
        1. Use Kubernetes/Submariner built-in metrics
          controller to publish load (RPS) to the Broker
        1. Use Kubernetes/Submariner built-in metrics
          controller to publish capacity (RPS) to the Broker

    1. Load distribution:

        1. Each Endpoint object will have a weight property
           the Lighthouse plugin each cluster will be aware of these weights
           and will use a simple Weighted-Round-Robin load balancing algorithm
           to distribute the load accordingly
        1. The weights will be published from the Broker to
           the Clusters

- Notes

    1. We might need to extend the Endpoint object to include the weights params
    1. Needs to determine the API for sharing the collected
      metrics from each cluster

### Test plan

Besides implementing Unitests and E2E tests which will be
configured according to the load balancing algorithm
during the actual implementation process. We plan on
experimenting with as close to possible real-life
scenarios.

#### Parameters

1. Optimization Objective Variables
    1. RTT -> Latency:
       latency between Data Centers is usually
       [steady](http://kth.diva-portal.org/smash/get/diva2:1334280/FULLTEXT01.pdf).
       Unpredicted events both in the network layer (packet loss,
       rate-limiting, delays due to congestion, and more), as well
       as application layer and system-related events (killing
       /replacing containers, disconnecting cluster and nodes,
       application bugs more) can cause latency and connectivity
       issues. There are numerous models and
       distributions for generating such disruptions.
        1. Testing Tool - We plan to use [Pumba](https://github.com/alexei-led/pumba)
           which is a chaos testing tool for Docker containers. It allows
           performing network emulation and application disruptions.
    1. Pricing:
       One of the key objectives in this EP is to
       improve the price one pays for service to service
       communication. While measuring the actual pricing is a
       complicated function to build, of different variables such
       as egress/ingress traffic between in-zone,out-zones/cloud
       providers, load balancing pricing overhead (if traffic
       goes through a dedicated load balancer it could impose
       extra charges), adaptive pricing (more traffic the price
       gets lower per GB), VPC pricing, Firewalls, pre-defined
       discounts, additional overlay services around the
       networking layer and more.
       [1](https://cloud.google.com/vpc/network-pricing),
       [2](https://cloud.google.com/products/calculator),
       [3](https://cloud.google.com/api-gateway/docs/pricing),
       [4](https://www.cloudmanagementinsider.com/data-transfer-costs-everything-you-need-to-know/),
       [5](https://aws.amazon.com/blogs/aws/aws-data-transfer-prices-reduced/).

1. Optimization Objective Constraints

    1. Capacity:
      Our main constraint to the optimization
      problem. Generally speaking, we would like to measure it
      by RPS. We plan to deduce such capacity according to
      the pod count of each service in each cluster. As we are
      aware that enforcing capacity in such a way is kind of
      stepping into Service Mesh territory there are such
      [precedent cases](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/2004-topology-aware-subsetting)
      in the Kubernetes community.
    s1. Load:
      Testing load on web-based systems is usually
      measured in users (e.g a demand for `x` users at a time `t`).
      A system is configured to hold an `x` amount of users at
      normal time and usually has extra resources to hold more.
      The load testing is derived from this configuration. We
      determine the amount of RPS that an average user
      generates, and deduce how our system will handle a certain
      amount of users. The load usually does not change drastically
      (unless under attack) in short time frames
      (minutes) and is predictable. [1](https://www.sciencedirect.com/science/article/pii/S0169023X06000954?casa_token=DiZI9NvfuPsAAAAA:4ziNfuft1UXUQf4-U678W1mft8_WXtpKKW-w_ZRh3BIqrB3rbJLBsb8XJkN5hTRc8FBVfxFzCw),
      [2](https://ieeexplore.ieee.org/abstract/document/911377),
      [3](https://ieeexplore.ieee.org/abstract/document/4308827)
      To measure different load situations, we will need to
      generate load for different time frames.
      As our system will probably update once every 5-20
      minutes, changes in load within these time frames
      are not relevant to our benchmark case.
      We are going to test the system under different loads in comparison of
      the system load capacity:

        1. low volume - low number of users
        1. benchmark volume - average (regular) number of users
        1. load test - the maximum number of users
        1. test destructively - simulate attack - much over then
          the maximal number of users the system suppose to handle

        Users load simulation can use Load Testing tool such as
        [lucust](https://locust.io/) /
        [Jmeter](https://jmeter.apache.org/),
        where one just configure the desire user behavior to test...

        From the above explanation, we can deduce that the load is a known number.
        We can add some randomness to it by choosing values from a
        given set or range at random, mainly
        according to the exponential and the heavy-tailed (Pareto
        and such) distribution models
        [1](https://ieeexplore.ieee.org/abstract/document/769420),
        [2](https://ieeexplore.ieee.org/abstract/document/7996729),
        [3](https://dl.acm.org/doi/pdf/10.1145/346000.346004).

1. User Location Distribution - should be picked at random
   from a normal distribution of the given user count
   [Twiter User Goe Distribution](https://www.researchgate.net/publication/315074173_Strategies_for_combining_Twitter_users_geo-location_methods)

1. Topology:

    1. Clusters Distribution:

        1. Cloud Provider
        1. Regions/Zones
        1. Applications:

    1. Clusters Layout:

        1. Standard web apps (API based)
        1. CDN and Gaming apps, high data volume
        1. Data pipeline apps

We shall compare different applications to the best of our knowledge.
And would like to invite you and suggest layouts for different  applications.
The rest of the testing parameters (load/capacity/random
latency) will be determined using randomize tools.

## Open questions

- How often do deployments affect changes in the application topology?
    1. How often deployment changes service placement
    1. How will it affect the optimization?
- How adaptive would we like to be for changes in the topology?
    1. We might need to trigger data collection and optimization after a new deployment.
- What data/metrics should we share through the Broker?
- What should we collect from the local cluster perspective?
    1. Pricing will have to be shared because they depend on the location/provider.
    1. Latency and RTT and Load should be collected from a local perspective.
- Cost Parameters/Metrics
    1. How often would we like to collect the parameters?
    1. Would we like to make the collection configurable?
- How can we understand what cloud provider the cluster is on.
- What cloud providers do we support?
- Real-life testbed
- More application topologies

## Implementation History

Currently, the Lighthouse DNS plugin within Submariner supports Round-Robin load
balancing between Exported/Imported service in the multi-cluster deployment.

## Future

### Extensions

1. Cost

    1. Parameters -
       we might want to think of more cost parameters to improve our model

        1. Network congestion
        1. Turnaround time - is the total time that it takes
           between the submission of a request and the return of
           the complete response to the customer. This is the
           main metric for users experience we would like to
           optimize it, as it derived from many of the variables
           we mentioned above (latency/RTT/Application) on one
           hand, but from the other, we are not sure yet, on how
           to fit it in our one-hop model, we would tackle this
           parameter in the future.

    1. Functions - we might want to consider and test different
       functions for combining the different parameters in the function

### Challenges

- Scalability to 5G – thousands of clusters
- Autoscale utilization
- Multi-hop
