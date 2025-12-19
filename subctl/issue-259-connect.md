# Provide a `connect` operation combining `deploy-broker` and `join`

## Summary

For end users, the purpose of Submariner is to connect clusters to
each other, enabling communications between them and providing
additional services on top. The fact that a broker, or hub cluster, is
involved, is a secondary concern; however that’s the first step
involved in setting up Submariner with `subctl`.

This enhancement proposal describes a `connect` subcommand which
configures two or more clusters together, without the separate
`deploy-broker` and `join` steps, or the use of a `broker-info.subm`
file.

## Proposal

A new `connect` operation, taking two or more Kubernetes contexts,
configures the corresponding clusters so that they are connected,
where possible and where this doesn’t result in conflicts with
existing configuration on the clusters. The contexts are specified
using the existing patterns in `subctl`: `--contexts` followed by a
comma-separated list of context names.

## Design Details

### Base scenario: no existing Submariner deployment

When the contexts point to clusters with no existing Submariner setup,
`connect` chooses one of the clusters to host the broker, deploys it
there, and connects all the clusters using the newly-deployed broker.

Broker selection should check for connectivity between the clusters:
the broker should only be installed on a cluster which is accessible
from all others. A `--broker-context` flag is provided to allow broker
selection by the user.

**No** `broker-info.subm` file is produced; see subsequent sections
for scenarios describing deployments with existing brokers.

`connect` supports all the existing options on `deploy-broker`,
including component selection, Globalnet configuration etc., apart
from those related to `broker-info.subm`. Since a single invocation of
`connect` has knowledge of more than one cluster, some additional
checks can be performed; in particular, the cluster CIDRs can be
checked for overlaps, with the following behaviour:

* in the default setup (no Globalnet configuration), Globalnet is
  automatically enabled if the CIDRs overlap;
* if Globalnet is explicitly disabled, and the CIDRs overlap,
  `connect` fails with an appropriate error.

Whether the broker is chosen automatically or specified using
`--broker-context`, if it is inaccessible from any of the clusters,
`connect` fails with an appropriate error, without changing any of the
configuration on any of the clusters.

### Connecting to an existing Submariner deployment

If any of the provided contexts point to a cluster with an existing
Submariner deployment, all such contexts are used to find the
corresponding brokers. If more than one broker is found, `connect`
fails with an appropriate error.

If only one broker is found, connection continues using that broker.
If the resulting setup is determined to be broken (Globalnet disabled
with a CIDR conflict involving one or more of the additional
clusters), `connect` fails with an appropriate error.

If the broker is inaccessible from any of the additional clusters,
`connect` fails with an appropriate error, without changing any of the
configuration on any of the clusters.

If all the provided contexts are already connected, `connect` doesn’t
do anything and exits successfully.

If a broker is found indirectly (that is to say, it isn’t listed in
the contexts to be connected, but is found through the BrokerK8s field
in the Submariner resource of an existing deployment), and the cluster
hosting it isn’t connected, that cluster’s configuration isn’t
changed. This allows hub-spoke style setups, with a broker that isn’t
part of the connected clusters, to be extended using `connect`. If the
broker’s context is listed in the contexts to be connected, the
cluster is connected if necessary.

### Detailed operations

The above considerations result in this sequence of steps:

1. Connect to all the provided contexts, looking for Submariner
   deployments. If a Submariner broker is found, remember it; if a
   Submariner deployment is found, determine the corresponding broker
   and remember it.

2. If more than one broker is found in step 1, fail.

3. If no broker is found in step 1, run connectivity tests from each
   cluster to all the other clusters’ API server. If any cluster is
   not accessible from another cluster, remove it from the set of
   broker candidates. If the set of broker candidates ends up empty,
   fail.

   This is liable to prove unwieldy for larger numbers of clusters;
   the test should be capped to a limit to be determined (let’s start
   with 4 or fewer clusters). If more clusters are specified,
   `connect` should ask the user to choose a broker with
   `--broker-context` and fail. (Alternatively, if `subctl` is used
   interactly, the user could be asked using the same prompting
   mechanism as labeling gateways.)

4. If a broker is found in step 1, run connectivity tests from each
   additional cluster to its API server. If any of the tests fail,
   fail.

5. Check for overlapping CIDRs among the clusters, including clusters
   already connected to the broker. If any overlap:

   1. If an existing broker was found, and Globalnet isn’t enabled,
      fail.
   2. If Globalnet was explicitly disabled, fail.
   3. Enable Globalnet.

   This entire step can be skipped when connecting to an existing
   broker which has Globalnet enabled.

6. If necessary, deploy the broker.

7. Join all the clusters.

## Backward Compatibility

`deploy-broker` and `join` are preserved as-is.

### Alternatives

None.

## External Dependencies

None.

## User Impact

No change is required. Existing users will be able to simplify
deployments relying on `deploy-broker` and `join`.
