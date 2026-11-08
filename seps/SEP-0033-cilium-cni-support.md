---
sep: 33
title: "Cilium CNI Support"
status: draft
authors: [kurtyoon]
created: 2026-10-13
components: [submariner, submariner-operator, subctl, shipyard]
repos:
  - submariner-io/submariner
  - submariner-io/submariner-operator
  - submariner-io/subctl
  - submariner-io/shipyard
  - submariner-io/submariner-website
---

# Cilium CNI Support

Tracking issue: [submariner#3168](https://github.com/submariner-io/submariner/issues/3168).
Earlier design and prototype by sanek9: [submariner#4105](https://github.com/submariner-io/submariner/issues/4105),
[submariner#4106](https://github.com/submariner-io/submariner/pull/4106),
[submariner-operator#4146](https://github.com/submariner-io/submariner-operator/pull/4146),
[subctl#1835](https://github.com/submariner-io/subctl/pull/1835).

## Summary

This proposal adds support for clusters that run Cilium with kube-proxy replacement, BPF masquerading and
eBPF host routing. Today such a cluster is treated as a `generic` network plugin and cross-cluster pod
traffic fails.

The proposal is smaller than the two approaches discussed on #4106. It adds an operator discoverer for
Cilium and a route-agent handler that keeps remote cluster CIDRs in Cilium's ip-masq-agent list. It also
gives `subctl` a short list of Cilium settings to check, and documents a limitation for exported ClusterIP
services on Cilium clusters. That limitation needs a change outside the CNI integration, which I will
propose separately for Globalnet clusters.

## Motivation

Issue #3168 has been open since 2024 and the maintainers have asked for a contributor more than once. At
least one user moved to Calico because of it
([comment](https://github.com/submariner-io/submariner/issues/4105#issuecomment-5913719417)), and the same
user, jdaln, has since offered to help with an implementation on #3168 and #4105. In November 2026 yboaron
described the masquerading conflict on #4247 as a known limitation and said that contributions for Cilium
support are welcome.

Cilium can be configured by hand so that pod traffic to another cluster is not masqueraded. The guide yboaron
points to on #4247 does it on Cilium 1.16 with the chart defaults, so with kube-proxy still running and no
BPF masquerading, by setting `enable-ipv4-masquerade` to false, after which `subctl verify` passed. That
switches masquerading off for every destination, which a cluster that relies on node SNAT for pod egress
cannot do, and the kube-proxy setup it covers is not the one this proposal is about. With kube-proxy
replacement and BPF masquerading, every manual variant needs the other clusters' ranges in advance: an
ip-masq-agent list that names them, an `ipv4-native-routing-cidr` wide enough to contain them all, or BPF
masquerading turned off, which drops Cilium back to legacy host routing on every node (see Alternatives).
Those ranges change whenever a cluster joins or leaves the clusterset, with Globalnet they are allocated at
join time, and the setting lives on every Cilium cluster in the set, so each change is an edit on each of
them, and a missed edit shows up as pod traffic leaving with a node address. The handler in this proposal
keeps the ip-masq-agent list current from the `Endpoint` objects route-agent already watches, which is the
one part a document cannot do. The prerequisites that do not change at runtime stay as documentation and
`subctl` checks.

The #4105/#4106 attempt stalled on a design question. Submariner could impersonate a ClusterMesh peer,
program Cilium's BPF maps, or do something smaller. The maintainers asked for a proposal that compares
these options and says what the project would have to maintain.

### Why the generic path fails

The generic route-agent path assumes that pod traffic goes through the node's routing table and netfilter
hooks. With eBPF host routing that is no longer true for traffic from pods. I checked the following against
Cilium 1.20.2 and reproduced each of them on kind.

1. A pod packet to a remote cluster is resolved by a FIB lookup in `bpf_lxc` and redirected straight to
   the egress device (`bpf/bpf_lxc.c:1343-1372`). Host routes and ip rules are still honoured, so
   Submariner's routes work. The node's netfilter rules, MSS clamp and conntrack never see the packet.
2. A remote CIDR resolves to the `world` identity. BPF masquerading rewrites world-bound pod traffic on any
   device that runs Cilium's `cil_to_netdev` program, unless the destination is in the native-routing CIDR
   or the ip-masq-agent list (`bpf/lib/nat.h:639-787`). Submariner's netfilter rules cannot pre-empt it.
3. Cilium's device auto-detection picks up `vx-submariner`
   (`pkg/datapath/linux/devices_controller.go:624-713`). On kind, every node added it within seconds of the
   device appearing, and pod traffic left with the node's 240.x VTEP address. This is the "240.x source"
   symptom reported on #4106.
4. Traffic that reaches the gateway from `vx-submariner` or from the cable goes through the host stack, so
   the gateway's netfilter rules, including Globalnet, keep working for pods on other nodes.

Submariner adds two gaps of its own. The generic discoverer reads the pod CIDR from `--cluster-cidr`
flags (`pkg/discovery/network/generic.go:174-199`), which may not exist with kube-proxy replacement and
cluster-pool IPAM. And service discovery hands clients a remote ClusterIP, or with Globalnet the global IP
of an `externalIPs` Service, leaving the translation to kube-proxy on the receiving gateway.

## Design

### Scope

IPv4 is the verified path. The handler treats IPv6 CIDRs the same way, but IPv6 BPF masquerading is beta in
Cilium and IPv6 is not part of Done When.

Access to an exported ClusterIP service whose backends run on a Cilium cluster is a documented limitation,
with and without Globalnet. With kube-proxy replacement, a request that arrives through the cable is not
translated on the gateway, so it does not reach a backend. The Cilium-side options that were tried either
hide the client's address from backends or need a hand-tuned MSS clamp (see Globalnet below).

Headless exports are not affected and are the workaround. For Globalnet clusters, a follow-up proposal
removes the limitation by translating ClusterIP exports in Submariner's own rules on the exporting gateway
(see Follow-ups). Clusters without Globalnet keep the headless workaround.

Cilium clusters that keep kube-proxy and masquerade with iptables, with BPF masquerading off, are also outside
this proposal. They keep today's generic path, and the handler stays idle there, because Cilium's
ip-masq-agent exists only in the BPF implementation. `subctl diagnose` reports that masquerading is handled
by iptables and points to the existing options: an `ipv4-native-routing-cidr` wide enough to cover the remote
ranges, `egress-masquerade-interfaces` limited to the uplink, which helps with the WireGuard and VXLAN cables
but not with libreswan, where traffic to the cable leaves through the uplink, or masquerading turned off.

### Datapath

Cluster A has pod CIDR 10.1.0.0/16 and cluster B has 10.2.0.0/16. `W` is a worker and `G` the active
gateway. "Stack" means the host network namespace with its netfilter hooks; "BPF" means a Cilium program
handled the hop.

| Hop | Request | Reply |
|---|---|---|
| `W_A` pod | BPF: FIB lookup, route `10.2.0.0/16 via 240.x dev vx-submariner` | BPF: overlay decap to the pod |
| `W_A` `vx-submariner` | VXLAN encap, no SNAT (10.2.0.0/16 is not masqueraded) | |
| `G_A` | stack: ip rule 150, table 150, MSS clamp, cable | stack: decap, MSS, route via `cilium_host` |
| `G_B` | stack: decap, MSS, `cilium_host`, then BPF overlay | stack: table 150, MSS, cable |
| `W_B` pod | BPF: overlay decap, source identity `world` | BPF: FIB to `vx-submariner`, no SNAT |

Every NAT in the design is netfilter on the active gateway. Both directions of a flow pass through that
gateway's stack, because packets reach it from a device rather than from a local pod. The exception is a
pod running on the gateway node itself. Its egress is redirected in BPF, which is handled by a
prerequisite below. With Globalnet, the gateway's SNAT and DNAT sit in the same place. The
[appendix](https://gist.github.com/kurtyoon/366ac0e42c60dfbd6b605fa030ffa382) has hop tables for Globalnet
egress and headless ingress.

### Discovery (submariner-operator)

A new `pkg/discovery/network/cilium.go` is appended after the KindNet discoverer, so it runs only when
OpenShift4, OVN-Kubernetes, Weave, Canal, Calico, Flannel and KindNet did not match, and before the generic
fall-through. A Calico cluster that also carries a `cilium` DaemonSet, during a migration or in chaining,
therefore stays Calico and keeps its IPPool handler. NotFound and Forbidden on the discoverer's lookups mean
no match, not an error, because a discoverer error aborts the reconcile.

It concludes `cilium` only when all of the following hold, and reports `generic` otherwise, with a log line
naming the step:

1. Exactly one DaemonSet whose pod template carries `k8s-app=cilium` has ready pods. None means no match;
   more than one means `generic`. The namespace comes from that DaemonSet, which covers a non-default
   namespace and a leftover `cilium-config` without an agent.
2. The cluster is not a managed Cilium distribution: no `anetd` DaemonSet (GKE Dataplane V2) and no
   `ipam: delegated-plugin` or AKS pod labels (Azure CNI Powered by Cilium). Both keep their own masquerade
   configuration, which the handler must not rewrite. Their markers come from the published manifests and
   are not verified on a live cluster.
3. `cilium-config` exists in that namespace, `cni-chaining-mode` is absent, `none` or `portmap`, and
   `cni-chaining-target` is not set. Any other chaining mode means another CNI owns pod networking.
4. The agent's `desiredNumberScheduled` covers every Linux node; otherwise `generic` with a warning.

Only after that, when IPAM is `cluster-pool` and the generic sources are empty, it reads the pod CIDR from
`cluster-pool-ipv4-cidr` and `cluster-pool-ipv6-cidr`, splitting on spaces and commas. In chaining mode
`ipam.mode` can stay at its `cluster-pool` default while another CNI allocates the addresses, so the chart
still renders `cluster-pool-ipv4-cidr` as `10.0.0.0/8`; reading it before the chaining check would advertise
that unused range. It also keeps `routing-mode`, `kube-proxy-replacement`, `enable-bpf-masquerade`,
`enable-host-legacy-routing`, `enable-ip-masq-agent` and `devices` in the operator's `ClusterNetwork`, where
the namespace env and the follow-up proposal's default come from; diagnose does not read them, it asks the
agent.

Cilium falls back to legacy host routing, with only an info log, when kube-proxy replacement is off or
masquerading runs in iptables rather than BPF. Diagnose therefore reads the effective mode from the agent
rather than from the ConfigMap.

`Submariner.Status.NetworkPlugin` becomes `cilium`. The namespace and the settings travel in
`ClusterNetwork.PluginSettings`, a field the operator declares today but never fills, so no CRD field is
added; Helm 3 does not upgrade `crds/`. The route-agent DaemonSet builder gains the namespace from the
discovered network and sets `SUBMARINER_CILIUM_NAMESPACE`, with an explicit envconfig tag, only when the
plugin is `cilium`: the route-agent DaemonSet rolls with `maxUnavailable: 100%`, so an env var added on
every cluster would restart every route-agent at once on operator upgrade.

OpenShift with Cilium is out of scope. The OpenShift4 discoverer copies `spec.networkType`, so such a
cluster reports `Cilium` today, a name the kube-proxy handler does not register for, which leaves it without
a Submariner datapath. After this change the name is in the plugin list, so that cluster gets the kube-proxy
handler, while the Cilium handler stays idle because the OpenShift branch sets no namespace. Running the
Cilium checks on that branch is a follow-up.

### route-agent Cilium handler

A new handler in `pkg/routeagent_driver/handlers/cilium` is registered for Cilium only. It is additive, like
the Calico IPPool handler. The existing handlers stay active. The MTU handler and the health checker
register for any plugin, but the kube-proxy handler, which owns the generic datapath, registers for
`cni.GetNetworkPlugins()`, and the event registry drops a handler whose plugin set lacks the current plugin,
so `cilium` goes into that list; a constant alone would cost a Cilium cluster that runs today as `generic`
its whole datapath at upgrade. A unit test asserts the handler set for `cilium`, and that `calico` and
`OVNKubernetes` do not get the Cilium handler.

Its job is to keep every remote cluster CIDR in Cilium's ip-masq-agent list. With ip-masq-agent enabled
and no ConfigMap, Cilium uses a built-in list that covers all of RFC 1918 and the reserved ranges
(`pkg/ipmasq/ipmasq.go:26-45`). That stops masquerading toward every private address outside the
cluster, not only toward remote clusters. On kind it broke pod connections to the broker API server: the
replies to the pod IP followed Submariner's route and were dropped on the far gateway. An explicit list
avoids that, so the proposal supports only an explicit list.

The desired set is the subnets of every remote `Endpoint`, kept current from endpoint create, update and
remove events. If ip-masq-agent is enabled but the `ip-masq-agent` ConfigMap does not exist, the built-in
list is in effect, so the handler logs it and diagnose fails, naming the Helm value
`ipMasqAgent.config.nonMasqueradeCIDRs`. The handler does not create the ConfigMap itself, because a
`create` permission cannot be limited by resource name.

When the ConfigMap exists, the handler writes `(list − owned) ∪ desired`. `owned` is the set it wrote last
time, kept in the annotation `submariner.io/non-masquerade-cidrs`, so user entries are never touched. It
also watches the ConfigMap, so a Helm upgrade that re-renders the list gets repaired without waiting for
an endpoint event. Cilium reloads the file on change, and propagation is the kubelet ConfigMap sync, about
a minute.

Only the active gateway writes, with conflict retries. A gateway that steps down removes nothing, since
removing entries would open a masquerade window. Uninstall removes only the annotated entries and the
annotation.

The handler does no I/O in its constructor. `Init` logs and never returns an error when the namespace, the
ConfigMap or the permission is missing, because an `Init` error is fatal for route-agent on every node;
without a namespace the handler stays idle. It writes only when `enable-ip-masq-agent`,
`enable-bpf-masquerade` and IPv4 or IPv6 masquerading are all on; otherwise it stays idle and diagnose
names the setting in effect. `Uninstall` runs on every node without gateway state, so it is idempotent and
retries on conflict. Each endpoint event recomputes the desired set from all remote Endpoints, so a remote
gateway failover never drops a CIDR.

The route-agent ClusterRole already has `get` and `list` on all ConfigMaps. The new rule adds `watch` and
`update` on `configmaps` with `resourceNames: [ip-masq-agent]`; a name-restricted watch is authorized only
with the field selector `metadata.name=ip-masq-agent`, so the informer uses it. A Role in the Cilium
namespace would have no creator on the Helm and OLM paths. The handler does not touch ipcache,
`cilium-config`, `CiliumNode` or any BPF map. `ipv4-native-routing-cidr` is not an alternative to the
handler, since it is a single static prefix and remote CIDRs change at runtime.

### Prerequisites checked by `subctl`

Remote CIDRs must not be masqueraded on any device that carries them. The recommended way is
ip-masq-agent with an explicit list (`ipMasqAgent.enabled=true` and `ipMasqAgent.config.nonMasqueradeCIDRs`).
The list holds the cluster CIDRs known at install time, plus `240.0.0.0/4` while `vx-submariner` is a
Cilium device, and the handler adds remote CIDRs later. Destinations outside the list are masqueraded as
before.

The other way is a pinned device set such as `devices: "eth+"`, so that no Submariner device runs Cilium's
programs. This only holds while the setting stays pinned, because Cilium attaches to devices at runtime
and `vx-submariner` is created after the agent starts. Pinning is still worth doing alongside the
ip-masq-agent list. Otherwise Cilium takes `vx-submariner`'s address as a NodePort address and applies
host-firewall policy to it.

The third prerequisite concerns pods on gateway nodes. Their egress is redirected in BPF along table 150
and skips the gateway's netfilter hooks. With libreswan this also skips the IPsec output policy and the
packet is dropped. With Globalnet, on any cable, it skips the Globalnet SNAT, and a backend's reply skips
the DNAT reversal. WireGuard without Globalnet happens to work.

The fix is legacy host routing on gateway nodes only, through a `CiliumNodeConfig` with
`enable-host-legacy-routing: "true"` and `nodeSelector: submariner.io/gateway=true`. It fixed every
gateway-node case on kind, including Submariner's own pods. The setting applies when the Cilium agent
starts. Every node that can become a gateway must therefore be labelled before Cilium starts there, and
labelling a node later needs an agent restart. Keeping workloads off gateway nodes with a taint is an
untested alternative.

`subctl join` warns when it cannot confirm the masquerade prerequisite. `subctl diagnose` checks the
masquerade prerequisite, the device set and the gateway-node routing, reading the effective device list and
host routing mode from the agent through pod exec, as the OVN check does for its database version.

One incompatibility is documented rather than worked around. With Cilium's own IPsec encryption enabled,
Cilium drops raw ESP from sources that are not Cilium nodes, which includes a remote Submariner gateway.
libreswan then needs forced UDP encapsulation (`--force-udp-encaps`, not verified), or another cable
driver.

### Globalnet

Egress through the cluster and namespace `GlobalEgressIP`s, headless-service ingress and the gateway
health check work unchanged, because they are netfilter on the gateway and both directions pass through
it. On kind, with overlapping CIDRs, they worked for clients and backends on both node types, with WireGuard
and libreswan, once the prerequisites were in place. Servers saw the client cluster's egress IPs. The
pod-selector `GlobalEgressIP` uses the same path but was not run.

ClusterIP exports do not work, for the reason given in Scope. The Cilium-side options that were tried
either hide the client's global IP from backends or need a hand-tuned MSS clamp. They are not part of this
proposal; the follow-up proposal moves the translation into Submariner's own gateway rules instead.

### Policy and identity

Cross-cluster traffic keeps the `world` identity, including Globalnet-translated traffic. Clusters with a
default-deny `CiliumNetworkPolicy` need `world` or CIDR rules for it. A `CiliumEgressGatewayPolicy` that
covers a remote CIDR must list it in `excludedCIDRs`. With the Cilium host firewall, node policy must allow
UDP 4800 between nodes and the cable ports from remote gateways. All of this is documented, not changed.
Giving remote clusters a Cilium identity is what the ClusterMesh-shaped publisher would add; see
Follow-ups.

### subctl

When the plugin is `cilium`, `subctl join` warns if neither the ip-masq-agent list nor a pinned device set
is configured. `subctl diagnose cni` accepts either, as join does: it checks that ip-masq-agent is enabled
with a ConfigMap and that every remote CIDR is in the effective list, or that the device set is pinned. It
also checks pods on gateway nodes with BPF host routing, and Cilium IPsec together with libreswan. `cilium`
is added to the plugin lists in diagnose (`supportedNetworkPlugins`) and gather (`networkPluginCNIType`);
without that, diagnose fails with "unsupported CNI plugin" and gather drops the iptables and nftables dumps
that Cilium clusters get today as `generic`. The Cilium checks use the pod-exec helper in `internal/pods`
and leave the OVN check untouched.

### Security model

No Cilium control plane spans clusters, and no Cilium trust relationship is created. That answers the
concern raised on #4106 that a ClusterMesh relationship is bidirectional. Remote clusters stay `world` to
Cilium policy. The handler's only write is one ConfigMap under a name-scoped role. That grant is bound
cluster-wide, so it also covers a `kube-system/ip-masq-agent` ConfigMap on clusters that do not run Cilium,
such as GKE's or the kubernetes-sigs agent's; the handler is not registered there, so nothing writes it,
but the permission exists. The route-agent ServiceAccount is shared by all nodes, but route-agent already
runs with host networking and `NET_ADMIN`, which is the same trust level.

### Cilium versions and maintenance

The design depends on four pieces of Cilium configuration: `enable-ip-masq-agent` and its file format,
`devices`, `enable-host-legacy-routing`, and `CiliumNodeConfig` (`cilium.io/v2`, since 1.16). That makes the
supported range Cilium 1.16 to 1.20. No BPF map layout, kvstore format or identity number is
used, so a Cilium upgrade needs no Submariner change unless one of those surfaces changes. The e2e job in
Step 5 runs a matrix of the two latest Cilium minors.

There are no new Go dependencies. The handler is one reconcile loop, and the discoverer and diagnose
checks follow the existing Calico and OVN code. The project would maintain one e2e job with two matrix
entries and one docs page.

### Alternatives Considered

**ClusterMesh-shaped ipcache publisher (#4106).** route-agent would run an embedded etcd or kine on every
node and publish remote CIDRs in the ClusterMesh kvstore format, so that Cilium tunnels worker traffic to
the gateway itself. It avoids masquerading in tunnel mode, handles gateway-node pods and gives remote
clusters an identity for policy. But it adds etcd or kine and gRPC to route-agent, an embedded binary in the
tens of MiB. It needs a CA, a cluster id and name and a reserved peer, it conflicts with existing
ClusterMesh and KVStoreMesh setups, and it does nothing for native routing, discovery or Services. It is
kept as a possible Phase 2 for identity.

**Direct BPF map programming.** route-agent would write remote entries into Cilium's ipcache map. The map
is recreated on every agent restart without foreign entries (`pkg/ipcache/cell/cell.go:107-129`), and its
layout changes between versions. sanek9's labs for #4106 reached the same conclusion.

**Legacy host routing cluster-wide.** This makes the generic path work at a per-packet cost on every
node, but it does not stop masquerading on `vx-submariner`. It is kept as the documented fallback, and per
node as the gateway-node prerequisite.

**Documentation only.** A deployment page could describe the ip-masq-agent list, the pinned devices and the
gateway-node routing, and `subctl diagnose` could check them. That covers everything in this proposal except
the remote ranges, which change at runtime and would have to be edited by hand on every Cilium cluster after
every join or leave. The handler exists for that one list; the rest of the proposal is documentation and
checks already.

| | This proposal | ClusterMesh publisher | Direct BPF | Legacy routing |
|---|---|---|---|---|
| User setup | ip-masq-agent list or pinned devices; gateway node config | cluster id/name, peer Secret | none | one Helm value |
| Code to maintain | ConfigMap reconcile, discoverer, diagnose | etcd/kine, TLS, kvstore schema | per-version struct layout | none |
| Dependencies and footprint | none | etcd or kine, gRPC; tens of MiB | none | stack cost on every node |
| Cilium coupling | four config keys | kvstore contract, identity ranges | BPF ABI | one config key |
| Routing modes | designed for both; tested in tunnel mode | tunnel only | both | both |
| Remote identity in policy | `world` | cluster-scoped | arbitrary | `world` |

### Backward Compatibility

Other network plugins are unaffected: the discoverer runs after every existing one, and the handler
registers for `cilium` only. A Cilium cluster that already runs with legacy host routing and `generic`
discovery keeps working. After an upgrade it is discovered as `cilium`, its advertised subnets do not
change because the cluster-pool fallback runs only when every generic source is empty, and the handler
reconciles the ConfigMap if there is one. An operator that sends `cilium` to an older route-agent image
would lose the kube-proxy handler there, so the operator sends `generic` when `Spec.Version` is older than
the release that carries this change. An image set through `ImageOverrides` cannot be dated, so the
documentation says to override the operator together with route-agent. Route-agent is also changed to treat
a plugin name it does not know as `generic`; today only an empty name gets that default.

## Implementation Plan

### Step 1: Reproduce and decide (done)

The kind lab used Cilium 1.20.2 and Submariner 0.24.2 with libreswan and WireGuard, plus a second pair of
clusters with overlapping CIDRs for Globalnet. Unmodified, pod traffic left with the 240.x address. A
pinned device set fixed it, and so did ip-masq-agent. The built-in list broke pod-to-broker traffic, which
a later run with an explicit list did not. Legacy routing on gateway nodes fixed pods there. Against a
Calico cluster, pod traffic and headless exports passed both ways, and ClusterIP exports passed into the
Calico side. That needed Calico's node address detection kept off `vx-submariner`, which is not specific
to Cilium: [calico#14300](https://github.com/projectcalico/calico/pull/14300) adds the device to Calico's
default exclude list, and [submariner#4249](https://github.com/submariner-io/submariner/issues/4249) tracks
the Submariner side. A failover between two gateway nodes by label change lost about 4 s of pod-to-pod
traffic, on the pair without Globalnet. A later run on Submariner 0.25.0 and Cilium 1.20.4 with the same
prerequisites gave the same results: the explicit list was in effect in BPF, headless exports,
`GlobalEgressIP` and the reverse direction passed on both cables, and ClusterIP exports still failed. The
[appendix](https://gist.github.com/kurtyoon/366ac0e42c60dfbd6b605fa030ffa382) has the matrix and the source
references. Still to run are the full `verify --only connectivity` suite, network policy, the VXLAN cable,
native routing and cluster-pool IPAM.

### Step 2: Discovery

**Files:** `submariner-operator/pkg/discovery/network/cilium.go`, `network.go`,
`internal/controllers/submariner/route_agent_resources.go`; `submariner/pkg/cni/plugins.go`.

Detection in the order above, the cluster-pool fallback, the `cilium` entry in the plugin list, the
`PluginSettings` carry and the namespace env. Unit tests use fake DaemonSets and ConfigMaps for
cluster-pool, kubernetes and multi-pool IPAM, for chaining, and for the managed-distribution markers.

### Step 3: Cilium handler

**Files:** `submariner/pkg/routeagent_driver/handlers/cilium/`, `pkg/routeagent_driver/main.go`;
`submariner-operator/config/rbac/submariner-route-agent/`; the Helm charts.

The reconcile with annotation ownership and a ConfigMap watch. Unit tests cover a missing ConfigMap, one
without entries, one with user entries, annotation drift, conflict retry and uninstall. The charts need no
template change; their RBAC is regenerated from the operator at chart release.

### Step 4: subctl

**Files:** `subctl/pkg/diagnose/cilium.go`, `cni.go`; `cmd/subctl/join.go`.

The join warning and the diagnose checks, using the pod-exec helper in `internal/pods`; the OVN check keeps
its own exec code.

### Step 5: CI and docs

**Files:** shipyard (a kind Cilium deploy option), `submariner/test/e2e/dataplane`, submariner-website
(`operations/deployment/cilium.md`).

One e2e job on `submariner` runs Cilium ↔ Cilium, in tunnel mode by default, through a `cni: cilium` branch
in shipyard that leaves the defaults and the OVN path unchanged. Cilium ↔ Calico follows once shipyard can
mix CNIs in one job. In the dataplane suite, the existing headless Globalnet spec gets the `basic` label,
which adds its two specs to `basic-connectivity` on every Globalnet pair. The pod-to-service and
ClusterIP-export specs read the target cluster's `status.networkPlugin` and are skipped only when it is
exactly `cilium`; when the value cannot be read they run. Five of the six Globalnet groups use
GlobalServiceIP, so toward a Cilium target the skip also removes their egress-selector and host-network
cases. On a Globalnet pair the ClusterIP-export case comes back with the follow-up proposal's specs; without
Globalnet it stays skipped, as does the raw pod-to-service spec. The docs cover the prerequisites, the
policy notes, and the ClusterIP limitation with its workaround.

## Done When

```bash
subctl show networks | grep cilium
subctl diagnose cni
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg bpf ipmasq list
subctl verify --context cluster1 --tocontext cluster2 --only connectivity,service-discovery
```

- [ ] The pod-to-pod specs of `verify --only connectivity` pass Cilium ↔ Cilium in CI and Cilium ↔ Calico
  in a manual run, in tunnel mode with libreswan and WireGuard. Legacy routing is used on gateway nodes only.
- [ ] On a Globalnet pair, the headless spec passes, and the ClusterIP-export specs are skipped toward
  Cilium.
- [ ] `subctl diagnose cni` reports each prerequisite.
- [ ] A Cilium CI job runs on `submariner`, and the deployment docs are published.

## Follow-ups

A second proposal, Globalnet service ingress without externalIPs, which I will submit after this one, moves
ClusterIP exports on Globalnet clusters into Submariner's own rules on the exporting gateway, the way
headless exports are handled today, and turns on by itself on Cilium clusters with kube-proxy replacement.
Clusters without Globalnet would need a small ingress range advertised to peers. In my kind runs a
release-0.25 peer did not pick up a subnet added to an existing Endpoint, and rejected an Endpoint that
arrived before its Cluster without re-checking it; I will file both as issues, and that part waits for those
fixes. A later Phase 2 could give remote clusters a cluster-scoped Cilium identity, so that policy can select
them and Egress Gateway skips them; #4106 is the working reference for that.
