# Disable Intra-cluster Connectivity

## Overview

Submariner currently offers flexible configuration for its connectivity and service discovery components.  
Within the connectivity layer, it supports both:

- **Intra-cluster connectivity**: Traffic between non-gateway nodes and the active gateway node inside a cluster.
- **Inter-cluster connectivity**: Traffic between gateway nodes across clusters.

In some scenarios, it would be beneficial to deploy Submariner only for inter-cluster connectivity — running just the
inter-cluster tunnel components — without deploying intra-cluster routing.

This would enable additional integration use cases, for example:

- Deployments where an external SDN or routing system manages intra-cluster routing,
while Submariner focusing exclusively on inter-cluster routing.

---

## Proposal

Introduce a configuration option to control the deployment of intra-cluster routing components.

**User choices:**

1. Deploy both intra-cluster and inter-cluster connectivity (**default**).
2. Deploy only inter-cluster connectivity.

This requires splitting the existing **RouteAgent** functionality into two logical components:

- **Intra-cluster routing** – e.g., creating `vx-submariner` interface, routing from non-gateway nodes to the active  
  gateway, including health check to remote clusters' gateway.
- **Inter-cluster routing** – routing between gateway nodes, including inter-cluster health check.

---

### API Change

Add a boolean field to the Submariner CRD:

```go
// Disable intra-cluster connectivity routing.
//nolint:lll // Markers can't be wrapped
// +operator-sdk:csv:customresourcedefinitions:type=spec,displayName="Disable Intra-cluster Connectivity"
// +operator-sdk:csv:customresourcedefinitions:type=spec,xDescriptors={"urn:alm:descriptor:com.tectonic.ui:booleanSwitch","urn:alm:descriptor:com.tectonic.ui:advanced"}
DisableIntraClusterConnectivity bool `json:"disableIntraClusterConnectivity,omitempty"`
```

### Behavior

- **false** → Deploy both intra- and inter-cluster routing.  
- **true** → Deploy only inter-cluster routing.

---

### Implementation Details

The value of `DisableIntraClusterConnectivity` will determine **submariner-routeagent** DaemonSet provisioning.

- When `DisableIntraClusterConnectivity` is **false** → `submariner-routeagent` DaemonSet will be deployed on  
  **all nodes** in the cluster (current behavior).
- When `DisableIntraClusterConnectivity` is **true** → DaemonSet will be deployed **only on gateway nodes** (nodes labeled with `submariner.io/gateway=true`).

### Required Changes

- **submariner-operator** code that provisions the `submariner-routeagent` DaemonSet should be updated.  
  If `DisableIntraClusterConnectivity` is true, add the following node selector to the DaemonSet pod template:

```yaml
nodeSelector:
  submariner.io/gateway: "true"
```

- Addtionally, a new environment variable will be added to `submariner-routeagent` pods:  

```bash
SUBMARINER_DISABLE_INTRAROUTING
```

The routeagent pods should honor this variable:

- If set to `"true"`, the routeagent should **not** configure routing rules needed for intra-cluster connectivity  
  (e.g., creating `vx-submariner` interface).

---

### Summary Table

| DisableIntraClusterConnectivity | SUBMARINER_DISABLE_INTRAROUTING | DaemonSet pods should run on |
|---------------------------------|----------------------------------|------------------------------|
| true                            | true                             | Only gateway nodes           |
| false                           | false                            | On all nodes                 |
