---
sep: 11
title: "Opt-in reflection of imported services into cluster.local"
status: draft
authors: ["kekkker"]
created: 2026-06-04
components: ["lighthouse"]
repos: ["lighthouse", "submariner-operator"]
---

# Opt-in reflection of imported services into cluster.local

<!-- Tracking issue: https://github.com/submariner-io/lighthouse/issues/2197 -->

## Summary

Add an **opt-in** lighthouse-agent mode that, alongside the existing `clusterset.local` DNS integration, reflects each imported
service into a native `cluster.local` `Service` + `EndpointSlice` in the consuming cluster. This lets multi-cluster service
discovery work on clusters whose cluster DNS **cannot be configured** to host the Lighthouse `clusterset.local` forward stanza —
most notably managed Kubernetes offerings with an immutable, node-local DNS.

Default behaviour is unchanged. The feature is gated behind `SUBMARINER_REFLECT_CLUSTER_LOCAL` (default `false`).

## Motivation

Lighthouse resolves `*.clusterset.local` by injecting a forward stanza into the cluster's CoreDNS `ConfigMap`:

```text
clusterset.local:53 {
    forward . <lighthouse-coredns-cluster-ip>
}
```

This requires that (a) the cluster DNS config is editable, and (b) the resolver the pods actually use loads that config. On a
growing class of managed platforms neither holds:

- **EKS Auto Mode** bakes a node-local CoreDNS into every node, bound to the kube-dns Service VIP, with an Auto-Mode-managed
  **immutable Corefile that does not read the `kube-system/coredns` ConfigMap**. The stanza is written but never reaches the
  resolver answering the VIP, so `clusterset.local` queries are forwarded upstream and `NXDOMAIN`. (Confirmed in the field: the
  in-cluster CoreDNS *pods* resolve `clusterset.local` correctly when queried directly by pod IP; only the node-local VIP path
  is broken.)
- Similar lockdown trends exist on other managed/auto node runtimes.

On these platforms Lighthouse's only integration point is removed, and operators fall back to side-channels (a second CoreDNS
Service plus per-pod `dnsConfig` pointing at it) — fragile, and a per-workload change.

## Design

When `SUBMARINER_REFLECT_CLUSTER_LOCAL` is enabled, a new agent controller watches the imported `EndpointSlice`s the agent
already syncs into the local service namespace (labelled `endpointslice.kubernetes.io/managed-by=lighthouse-agent.submariner.io`,
carrying `multicluster.kubernetes.io/service-name`, `.../source-cluster`, and `lighthouse.submariner.io/sourceNamespace`).

For each imported slice whose source cluster is **not** the local cluster, it reconciles, in the service's namespace:

1. A headless `Service` named **`<service>-<sourceCluster>`** (`clusterIP: None`, no selector), ports copied from the slice.
2. A native `EndpointSlice` labelled `kubernetes.io/service-name=<service>-<sourceCluster>`, addresses/ports copied from the
   imported slice (the addresses Submariner's data plane already routes), associating it with the reflected Service.

Both are labelled `app.kubernetes.io/managed-by=lighthouse-clusterlocal-reflector` and are only ever created, updated, or
pruned by the reflector.

Workloads then resolve **`<svc>-<cluster>.<ns>.svc.cluster.local`** through the standard resolver — no second CoreDNS, no
per-pod `dnsConfig`. This is the same effect as Liqo service reflection, kept on Submariner's (auto-failover-capable) data plane.

The `<svc>-<cluster>` naming reuses the agent's existing per-cluster slice convention (imported slices are already suffixed with
the source cluster). It is what keeps reflection collision-safe and conformant-by-construction: it never squats a bare local
name the cluster owner controls, and two exporting clusters with the same `<ns>/<svc>` get **distinct** local names — the exact
ambiguity `clusterset.local` exists to prevent is sidestepped rather than papered over with a flat `cluster.local` record.

A reflected name that is not a valid DNS-1123 label (e.g. `<service>-<cluster>` exceeds 63 characters) is skipped and logged. A
pre-existing Service of the target name that the reflector does not own is left untouched (skipped and logged). The reflected
Service is pruned when the last imported slice for a `(service, cluster)` pair is removed.

### Alternatives Considered

- **Make Lighthouse serve under `cluster.local` by default** — violates the MCS-API (KEP-1645), which defines `clusterset.local`
  as the zone for imports, and breaks name-collision handling for the general multi-peer case.
- **External controller** — a standalone controller implementing this logic has run in production on an EKS Auto Mode ↔ Hetzner
  clusterset, fully replacing the side-channel. It works but is per-operator glue with its own image/RBAC; folding it into the
  agent gives every Submariner user a supported switch.
- **Fix the platform DNS** — not possible on immutable node-local resolvers.

### Backward Compatibility

Fully backward compatible. The feature is opt-in and default-off; when disabled there is no behaviour change and no new objects.
`clusterset.local` remains the default, MCS-conformant path and continues to work alongside reflection where DNS is editable.

## Implementation Plan

### Step 1: Agent flag and reflector controller

**Files:** `pkg/agent/controller/types.go`, `pkg/agent/controller/agent.go`,
`pkg/agent/controller/clusterlocal_reflector.go`

Add `ReflectClusterLocal` to `AgentSpecification` (env `SUBMARINER_REFLECT_CLUSTER_LOCAL`), threaded like `GlobalnetEnabled`.
Add `ClusterLocalReflector` (a resource syncer over imported `EndpointSlice`s) started only when the flag is set; reflect /
collision-guard / prune as above. Unit and two-cluster integration tests.

### Step 2: Operator support

**Files:** `submariner-operator` (Helm chart + RBAC)

Expose a `serviceDiscovery.reflectClusterLocal` value that sets the env var, and extend the lighthouse-agent ServiceAccount RBAC
with `services` and `endpointslices` create/update/delete (the reflector creates those).

## Done When

```bash
make unit
```

- [ ] `SUBMARINER_REFLECT_CLUSTER_LOCAL=true`: an exported service resolves at `<svc>-<cluster>.<ns>.svc.cluster.local` in the
      consuming cluster via the standard resolver.
- [ ] Flag unset: no `cluster.local` Service/EndpointSlice is created for imports.
- [ ] Reflected objects are pruned when the export is removed.
- [ ] Operator value + RBAC merged.
