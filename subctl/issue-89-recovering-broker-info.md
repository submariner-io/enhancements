# Recover broker-info.subm file from the running Submariner

Related Issue:
[Option for recovering the broker-info.subm file from a running broker cluster](https://github.com/submariner-io/enhancements/issues/143)

## Summary

`broker-info.subm` file is created when a broker is installed. The file contains information critical to have a functional environment.
There is a possibility that the file could be deleted or lost for one or the other reason. In such cases, no new clusters could be joined.

This proposal describes a way to recover the `broker-info.subm` file from the running broker.

## Proposal

A new `subctl` command will be added that would fetch needed data from the broker and Submariner and write it to `broker-info.subm` file.
The file would be Base64 encoded.

## Design details

* subctl command name: `recover-brokerinfo`
* Flags for the command:
  * All flags that `restConfigProducer` provides
    * `kubeconfig` to specify kubeconfig of the cluster in which Submariner is installed
* Data to be fetched from running broker or Submariner:
  * Broker URL
  * Client token
  * IPSec PSK
  * Components installed i.e. Submariner and Service discovery
  * whether service discovery component is installed
  * default custom domains

The last three information will be retrieved from the broker `Spec`, client token from the Broker admin service account
`submariner-k8s-broker-admin` and Broker URL and IPSec PSK from Submariner `Spec`.

If Submariner is not installed in the cluster specified by the `--kubeconfig` flag, end the execution and throw an error. The logic
first checks if Broker is installed on the same cluster as Submariner. If not, Broker cluster will be accessed from the Submariner cluster
via the Broker URL.

### Usage

The new command can be used as follows (assuming Submariner is installed in cluster2):

`subctl recover-brokerinfo --kubeconfig output/kubeconfigs/kind-config-cluster2`

### Alternatives

None that is automated. User can always manually keep a copy of `broker-info.subm` file.

## External Dependencies

None.

## User Impact

Users would be able to retrieve information in cases where generated `broker-info.subm` file is lost.
