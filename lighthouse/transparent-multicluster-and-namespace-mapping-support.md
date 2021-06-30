# Transparent multicluster support

## Summary

Lighthouse is a component from Submariner which can be used independently of the dataplane components,
and exchanges information in the form of ServiceImport and EndpointSlice CR/CRDs over the broker.

Lighthouse implements a CoreDNS plugin to perform dns resolution for clusterset services.

Lighthouse implements the
[MCS API](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api)
with some additional extensions.

In terms of DNS resolution, the MCS API specifies that multicluster services can be resolved with the following
FQDN format:

* $service-name.$namespace.svc.clusterset.local

In the context of [transparent multicluster](https://github.com/kcp-dev/kcp/tree/main/docs/investigations),
two extra features are necessary:

* The ability to resolve multicluster services via $service-name.$namespace or $service-name, in a way
  that a pod running on a different cluster would resolve such service as if it was a local service,
  hence removing the need to modify any deployment logic for the application.

* The ability to map namespaces, removing the namespace sameness requirement. This is necessary to cover
  namespace conflicts (where a pre-existing namespace exists in a target cluster), or multiple logical clusters
  using the same namespace under the same physical cluster.

## Proposal

This enhancement proposal defines a possible implementation for such features while retaining compatibility with the MCS API.

## Design Details

### Transparent service resolution

This feature should be disabled by default, and enabled by an operator flag.

Should be implemented in the lighthouse-coredns plugin, where support for non .svc.clusterset.local queries
would be responded when the corresponding ServiceImports and EndpointSlices exist.

The submariner-operator must take care of configuring the corresponding responders in the system CoreDNS config
or kubedns settings with fallback to the default kubernetes responders. In the case of OpenShift it should
create the right DNS entries for the OpenShift DNS operator. (needs investigation to make sure that we can
properly fallback to the normal kubernetes resolution).

### Namespace mapping

Namespace mapping could be performed by annotating a namespace with the target clusterset namespace, for example.

If we take this example, in the physical cluster A:

   kubectl annotate namespace foo multicluster.kubernetes.io/namespace=bar

the behavior described in the following sections will be implemented:

#### ServiceImports

Any ServiceImport targetted for the clusterset namespace "bar" would land in the namespace "foo".

#### ServiceExports

Any ServiceExport performed in the namespace "foo" will land in the clusterset namespace "bar" for other clusters / the broker.

#### DNS resolution from Pods

Pods from the namespace "foo" will resolve multicluster services in the following way:

* $service will resolve as $service.bar.svc.clusterset.local
* $service.foo will resolve as $service.bar.svc.clusterset.local
* $service.foo.svc.clusterset.local will resolve as $service.bar.svc.clusterset.local

This means that, when this mode is enabled, the lighthouse-coredns plugin will need to track
a list of pod-ip to namespace, so DNS requests can be traced back to namespaces. This
is something that the kubernetes coredns plugin does internally.

## Work items

* Enhancement proposal
* submariner-operator support for transparent resolution: bare CoreDNS / kubedns
* submariner-operator support for transparent resolution: OpenShift DNS operator
* lighthouse-coredns plugin support: transparent $service.$namespace resolver
* lighthoues-coredns plugin support: namespace mapping
