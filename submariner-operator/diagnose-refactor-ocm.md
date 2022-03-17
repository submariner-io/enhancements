# Refactor subctl diagonose for use with OCM

## Summary

Currently only way to run diagnostics on a Submariner installation is through `subctl` CLI. This requires user or admin
to run it manually from a host. There is no way to run it through UI like OCM or deployment tools like helm charts etc.

## Proposal

This enhancement proposes a means to run `subctl diagnose` through a `SubmarinerDiagnose` Job. User or Admin can deploy
this Job manually. In case of OCM, a new SubmarinerDiagnose Controller will be added to SubmarinerAddon which will deploy
this Job, track its completion and fetch results once done.

## Design Details

* A `submariner-diagnose` Job will be created which will run `subctl diagnose` with relevant flags as `command`. This
 job will run once to completion.
* A new `SubmarinerDiagnose` CRD will be added to SubmarinerAddon. OCM UI will create this CR with relevant values
 to trigger running diagnose from OCM UI.`
* A new `SubmarnerDiagnoseController` Controller will be added to OCM SubmarinerAddon. This controller will spawn the SubmarinerDiagnose
 Job when SubmarinerDiagnose CR is created, track completion.
* Only one active `submariner-diagnose` Job will be run in a cluster by SubmarinerAddon at a given time. Users can create
 multiple SubmarinerDiagnose Jobs manually and we won't enforce any checks to prevent that.
* When `submariner-diagnose` Pod is scheduled, Status field in SubmarinerDiagnose CR will be updated with name and namespace
 of the Pod for UI to use.
* Support will be added to OCM UI to run diagnostics on a cluster with a single click and then display the results. For now,
 it will run with sub-option `all`
* UI will provide a link to logs of `submariner-diagnose` Pod and show results as is to the user.
* Results of diagnose will also be exposed as Prometheus metrics. This will allow any Prometheus based observability
  solutions to consume the results of diagnose. This will also provide a quick and easy way to show results in the UI while
  SubmarinerAddon changes are in-progress.
* Advantage of this design is that newer features can be added to `subctl diagnose` without requiring corresponding changes
  in UI to consume those results.
* This will also simplify the code in SubmarinerAddon as it will no longer need to extract and parse diagnose results.
* [Initial scope] For initial implementation, SubmarinerDiagnose will always run with `all` but without Firewall Options.
 This is to reduce complexity on OCM UI side to pass credentials for remote cluster which are required for Firewall option.
 This support will be added in later release.

### Alternate Design

* A new option `--output json` will be added to `subctl diagnose` to capture output in a json format instead of current
  user readable text format.
* When `submariner-diagnose` Job is completed, SubmarinerDiagnoseController will fetch the logs from diagnose Pod, extract the
  results in json format and populate the relevant Status fields in SubmarinerDiagnose CR.
* This will allow UI to present results of diagnostic run in a more UX friendly way, but this will require much more
 effort in Addon and UI. For simplicity reasons this approach is not favored.

### SubmarinerDiagnose CRD

```Go
// SubmarinerDiagnoseSpec defines the desired configuration to run SubmarinerDiagnose
type SubmarinerDiagnoseSpec struct {
All             bool         `json:"all,omitempty"`
CNI             bool         `json:"CNI,omitempty"`
Connections     bool         `json:"connections,omitempty"`
Deployment      bool         `json:"deployment,omitempty"`
Firewall        bool         `json:"firewall,omitempty"`
GatherLogs      bool         `json:"gatherLogs,omitempty"`
K8sVersion      bool         `json:"k8sVersion,omitempty"`
KubeProxyMode   bool         `json:"kubeProxyMode,omitempty"`
FirewallOptions FirewallOptions `json:"firewallOptions,omitempty"`

// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
// Important: Run "make manifests" to regenerate code after modifying this file
// Add custom validation using kubebuilder tags: https://book-v1.book.kubebuilder.io/beyond_basics/generating_crd.html
}

type FirewallOptions struct {
InterCluster bool `json:"interCluster,omitempty"`
IntraCluster bool `json:"intraCluster,omitempty"`
Metrics      bool `json:"metrics,omitempty"`
RemoteCluster            string `json:"remoteCluster,omitempty"`
RemoteK8sAPIServer       string `json:"remoteK8sAPIServer,omitempty"`
RemoteK8sAPIServerToken  string `json:"remoteK8sAPIServerToken,omitempty"`
RemoteK8sCA              string `json:"remoteK8sCA,omitempty"`
RemoteK8sSecret          string `json:"remoteK8sSecret,omitempty"`
RemoteK8sRemoteNamespace string `json:"remoteK8sRemoteNamespace,omitempty"`
}

// SubmarinerDiagnoseStatus defines the observed result of SubmarinerDiagnose
type SubmarinerDiagnoseStatus struct {
Conditions []metav1.Condition `json:"conditions"`
K8sVersion string `json:"k8sVersion,omitempty"`
CNIType string `json:"cniType,omitempty"`
KubeProxyMode bool `json:"kubeProxyMode,omitempty"`
ConnectionsStatus ConnectionsStatus `json:"connectionsStatus,omitempty"`
FirewallStatus FirewallStatus `json:"firewallStatus,omitempty"`
// DeploymentStatus already captured in SubmarinerStatus, no need to duplicate information.
// DeploymentStatus SubmarinerStatus `json:"DeploymentStatus,omitempty"`
}

type ConnectionsStatus struct {
// TBD in future when UI supports Topology view
}

type FirewallPortStatus string

const (
Allowed = "allowed"
Blocked = "blocked"
Unknown = "unknown"
)

type FirewallStatus struct {
Metrics FirewallPortStatus `json:"metricsStatus,omitempty"`
VxLanTunnel FirewallPortStatus `json:"vxLanTunnel,omitempty"`
IPSecTunnel FirewallPortStatus `json:"IPSecTunnel,omitempty"`
}
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: submariner-diagnose-xyz123
spec:
  template:
    spec:
      containers:
      - name: submariner-diagnose-xyz123
        image: submariner-diagnose
        command: ["subctl",  "diagnose", "all", "--in-cluster"]
      restartPolicy: Never
      serviceAccount: submariner-diagnose
      serviceAccountName: submariner-diagnose
  backoff-limit: 0
```

### Prometheus Metrics

* submariner_diagnose_errors Total count of error detected, label: cluster-name: counter, will only go up, +1 for each
 error detected by running `subctl diagnose`
* submariner_k8s_version_support K8s version supported label: cluster-name(1 - for true, 0 - for false)
* submariner_cni_support Cni support . label: cluster-name, cni name (1 - for true, 0 - for false)
* submariner_globalnet_overlapping_CIDR Globalnet - is overlapping CIDR label: cluster-name(1 - true, 0 - false)
* submariner_kube_proxy_mode_support Kube proxy mode support label: cluster-name (1 - true, 0 - false)
* submariner_deployment_status Deployment status, label:cluster-name, daemonset / deployment name (1 - all good, 0 - error)
* submariner_pod_status Pod status, label: cluster-name, pod name, status ( 1 - all good, 0 - error)
* submariner_pod_restart_count Pod restart count (if count > 5), label: cluster-name, pod name, status . value = restart count
* submariner_firewall_vxlan_tunnel Firewall intra-cluster, label: cluster-name (1 - all good, 0 - error)
* submariner_firewall_ipsec_tunnel Firewall inter-cluster, labels: local-cluster, remote-cluster (1 - all good, 0 - error)

## Work items

* Add option for `--output json` to `subctl`
* Add updating Prometheus metrices to `subctl`
* Add SubmarinerDiagnose CRD to SubmarinerAddon
* Add SubmarinerDiagnoseController to SubmarinerAddon
* OCM UI changes to consume Prometheus Metrics generated by SubmarinerDiagnose
* OCM UI changes to consume SubmarinerDiagnose results
* CI
* Docs
