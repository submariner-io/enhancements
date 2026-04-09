# Submariner metrics redesign

Related Issue:
[Exporting Submariner metrics without needing to open ports](https://github.com/submariner-io/submariner/issues/1944)

## Summary

Submariner Gateway and Globanet pods run with HostNetworking.
To access metrics exported by these two we need to open ports in the infra firewall.
With the move towards minimizing changes needed in cloud infra for Submariner,
there is a use case to export metrics in a way that doesn't require any of these ports to be opened.
The current design also has a limitation where these ports might be in use by some other service,
causing a conflict and resulting in metrics becoming inaccessible.

## Proposal

This enhancement proposes a redesign of existing metrics for Submariner Gateway and Globalnet by adding a new metrics proxy
service. Pods running on the same node can access ports without requiring ports to be opened in the cloud infra.
Queries for metrics will reference the proxy service instead of the Gateway and Globalnet metrics ports directly,
and each proxy will provide access to the metrics on the same node.

This proposal also recommends changing the ports used by the metrics HTTP servers in the Gateway and Globalnet pods to a free range
port to avoid conflicts with ports already in use on the host network.

## Design Details

* A new DaemonSet `submariner-metrics-proxy` will be created. It will use the NodeSelector `submariner.io/gateway=true`
to run on all gateway nodes, active or not.
* The DaemonSet will also use tolerations to make sure it can always be scheduled on the gateway node, irrespective of any taints.
* The `submariner-metrics-proxy` Pod will use `netcat` to act as proxy and forward all incoming requests on metrics service port to the
metrics port on the node IP.
* The proxy pod will run two containers if Globalnet is enabled: one each for Gateway and Globalnet.
* The existing metrics service for Gateway and Globalnet will be modified to use the new metrics proxy pod. This will be
done by using same label selector as the metrics proxy DaemonSet. This will make sure users can continue using existing services
and ports without needing to make any changes themselves.
* `submariner-gateway` will be modified to use port `32780` instead of `8080` and `submariner-globalnet` will be
modified to use `32781` instead of `8081`. A new env variable will be used for these ports.
* [Optional] Modify `subctl cloud prepare` to no longer open metrics ports.

### Submariner Metrics DaemonSet

```yaml
---
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: submariner-metrics-proxy
spec:
  selector:
    matchLabels:
      app: submariner-metrics-proxy
  template:
    metadata:
      labels:
        app: submariner-metrics-proxy
    spec:
      nodeSelector:
        "submariner.io/gateway": "true"
      containers:
        - name: gateway-metrics-proxy
          image: quay.io/submariner/nettest:devel
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_IP
            valueFrom:
              fieldRef:
                fieldPath: status.hostIP
          - name: SUBM_GW_METRICS_PORT
            value: "32780"
          command: ["/usr/bin/nc"]
          args: ["-v", "-lk", "-p", "8080", "-e", "nc", "$(NODE_IP)", "$(SUBM_GW_METRICS_PORT)"]
        - name: globalnet-metrics-proxy
          image: quay.io/submariner/nettest:devel
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_IP
            valueFrom:
              fieldRef:
                fieldPath: status.hostIP
          - name: SUBM_GN_METRICS_PORT
            value: "32781"
          command: ["/usr/bin/nc"]
          args: ["-v", "-lk", "-p", "8081", "-e", "nc", "$(NODE_IP)", "$(SUBM_GN_METRICS_PORT)"]
      tolerations:
      - operator: Exists
      restartPolicy: Always
```

### Changes to Submariner Gateway Metrics service

```diff
metadata:
  name: submariner-gateway-metrics
  labels:
-   app: submariner-gateway
+   app: submariner-metrics-proxy
...
spec:
  selector:
-   app: submariner-gateway
+   app: submariner-metrics-proxy
```

### Changes to Submariner Globalnet Metrics service

```diff
metadata:
  name: submariner-globalnet-metrics
  labels:
-   app: submariner-globalnet
+   app: submariner-metrics-proxy
...
spec:
  selector:
-   app: submariner-globalnet
+   app: submariner-metrics-proxy
```

### Additional Details

These changes will be done in the following order to avoid any breakage in CI:

1. Modify `subctl diagnose` to actually query metrics rather than check for metrics port availability on firewall.
2. Create and deploy the new metrics proxy service. As part of this change the Gateway and Globalnet
DaemonSets will be modified to set the metrics port in an env variable. They will still use the current ports, `8080`
and `8081`. This change will be made in the `submariner-operator` repo.
3. Modify gateway and globalnet code to use the new metrics port env variable when creating their metrics HTTP servers. If not
available, it will default to the new values. This change will be in the `submariner` repo.
4. Modify the metrics port env variable to use new values. This change will be in `submariner-operator` repo.

### Pros

* No user facing changes to existing services.
* Only one extra pod created for both gateway and globalnet.

### Cons

* Adds extra pod(s).

### Backward Compatibility

`subctl diagnose` checks for metrics port in firewall. This test will need to be modified to not check for port itself
but try acessing the metrics irrespective of ports being open or not. This makes this change not backward compatible with
older versions of `subctl`.

Older versions of `subctl diagnose` will not work these changes. So users will have to make sure they're using compatible
version of `subctl`. The new `subctl diagnose` will work with older versions of subctl as `diagnose` changes will be fully
backwards compatible.

### Alternatives

* Run separate proxy for globalnet and gateway. This would add an extra pod without adding any value as those pods will
still need to run on the same node. Rejected because extra overhead without any value add.
* Create a single service for globalnet and gateway metrics. This would not be backward compatible because users using
existing service names to pull metrics will have to modify their setups.

## External Dependencies

None.

## User Impact

None.
