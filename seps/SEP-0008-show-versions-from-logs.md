# Extract version information from pod logs

Related issue:
[show versions should extract version information from pod logs](https://github.com/submariner-io/subctl/issues/798)

## Summary

`show versions` currently shows version of the `DaemonSet/Deployment` which is the version configured. But actual version
depends on the image running on the pods and can vary based on ImageOverrides, mirrors etc.

This proposal describes a way to show actual version running on the pods.

## Proposal

Along with the version configured in the `DaemonSet/Deployment` and Submariner CR, `show version` will also show actual
version running in the pods. `--version` flag will be added to all Submariner binaries. `subctl` will `exec` the binary
in each component's pod with `--version` to extract the version. All submariner components will also log the version
during bring up. If `subctl` is unable to run binary with `--version` for any reason, it will extract the version from
the logs.

## Design Details

### Add `--version` flag to all binaries

Add a `--version` flag to all submariner binaries. When executing the binary with this flag it will return the version
in the following format:

`component-name version: x.y.z`

### Capture version in pod logs

Add a log message to all submariner pods in the following format:

`component-name version: x.y.z`

The component name will be same as defined in `submariner-operator/pkg/names/names.go` e.g. For Submariner Operator
running version 0.15.0 this will be `submariner-operator version: 0.15.0`.

Following components already log version but format of log message will change:

* submariner-operator

Following components don't log version and will need changes to log message:

* submariner-gateway
* submariner-routeagent
* submariner-lighthouse-agent
* submariner-lighthouse-coredns
* submariner-globalnet
* submariner-metrics-proxy
* submariner-networkplugin-syncer

### Modify output of show versions

Output of `subctl show versions` will be modified to show version from pod logs or `--version` output as follows:

#### Current output

```bash
subctl show versions

Cluster "cluster1"
 ✓ Showing versions
COMPONENT                       REPOSITORY           VERSION
submariner-gateway              quay.io/submariner   0.15.1
submariner-routeagent           quay.io/submariner   0.15.1
submariner-metrics-proxy        quay.io/submariner   0.15.1
submariner-operator             quay.io/submariner   0.15.1
submariner-lighthouse-agent     quay.io/submariner   0.15.1
submariner-lighthouse-coredns   quay.io/submariner   0.15.1
```

#### New output

```bash
subctl show versions

Cluster "cluster1"
 ✓ Showing versions
COMPONENT                       REPOSITORY           CONFIGURED  RUNNING
submariner-gateway              quay.io/submariner   0.15.1      0.15.1
submariner-routeagent           quay.io/submariner   0.15.1      0.15.1
submariner-metrics-proxy        quay.io/submariner   0.15.1      0.15.1
submariner-operator             quay.io/submariner   0.15.1      0.15.1
submariner-lighthouse-agent     quay.io/submariner   0.15.1      0.15.0
submariner-lighthouse-coredns   quay.io/submariner   0.15.1      0.15.0
```

### Enhancing diagnose

`subctl diagnose` will also be enhanced to highlight mismatch between configured and running versions. This will not be
treated as an error as this might be intentional.

### Pros

* Provides more accurate information to user
* Easier to troubleshoot issues where actual image version is different from one configured.

### Cons

* Require changes to all components, not just `subctl`

### Backward Compatibility

* `show version` output will change, which may break any user scripts that rely on output in current format.
* Submariner Operator already logs the version but format of log message will change. This may also impact user automation.

### Alternatives

None

## External Dependencies

None

## User Impact

* Users will need to modify any scripts or tooling that rely on current output of `show versions` or version log message
in Submariner Operator pod.
