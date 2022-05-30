# Submariner VPC Peering Support

This document describes the technical details and impact of our proposal design
to include Virtual Private Cloud (VPC) Peering support for Submariner and
`subctl` located at this [Pull Request](https://github.com/submariner-io/cloud-prepare/pull/190).

## Abstract
This idea was born meanwhile the CockroachDB project focused to deploy a
multi-cluster setup of CockroachDB and communicate the different clusters across
regions using Submariner.

One of the requirements generated from that project was to deploy CockroachDB on
AWS in different regions at the same time, using VPC Peering and configure VXLan
(instead of IPsec) to increase network performance because VPC Peering
connections are already secure and encrypted.

In order to achieve that, we observed neither the Operator nor the `subctl` CLI
supported VPC Peering setup, configuration or deployment. For that reason, we
decided to provide such functionality to cloud-prepare and subctl.  With a set
of sub-commands in subctl, developers will automatically provision and configure
a VPC Peering between two clusters of the same cloud provider. Once
cloud-prepare allows VPC Peering, the Submariner Operator will include the VPC
Peering configuration as well.

## Design

The following sections include the technical aspects of this feature adding and
how it works.

### New `subctl` sub-commands

We purpose to add some subcommands to `subctl` to use the new VPC Peering
features described on this document.

#### VPC Peering Subcommands

This will be how new added sub-commands would be like.

```bash
$ ./bin/subctl cloud VPC Peering
This command creates a VPC Peering between different clusters of the same cloud
provider.

Usage:
  subctl cloud VPC Peering [command]

Available Commands:
  clean       Remove a VPC Peering between clusters
  create      Create a VPC Peering between clusters

Flags:
  -h, --help   help for VPC Peering

Global Flags:
      --kubeconfig string    absolute path(s) to the kubeconfig file(s)
      --kubecontext string   kubeconfig context to use

Use "subctl cloud VPC Peering [command] --help" for more information about a
command.
```

#### VPC Peering create/clean subcommand

```bash
$ ./bin/subctl cloud VPC Peering create
This command creates a VPC Peering between different clusters of the same cloud
provider.

Usage:
  subctl cloud VPC Peering create [command]

Available Commands:
  aws         Create a VPC Peering on AWS cloud
  gcp         Create a VPC Peering on GCP cloud
  generic     Create a VPC peering between generic clusters for Submariner

Flags:
  -h, --help   help for create

Global Flags:
      --kubeconfig string    absolute path(s) to the kubeconfig file(s)
      --kubecontext string   kubeconfig context to use

Use "subctl cloud VPC Peering create [command] --help" for more information
about a command.
```

#### VPC Peering cloud-provider option (Available for create & clean subcomands)

```bash
$ ./bin/subctl cloud
VPC Peering create aws --help
This command prepares an OpenShift installer-provisioned infrastructure (IPI) on
AWS cloud for Submariner installation.

Usage:
  subctl cloud VPC Peering create aws [flags]

Flags:
      --credentials string           AWS credentials configuration file (default
"/home/user/.aws/credentials")
  -h, --help                         help for aws
      --infra-id string              AWS infra ID
      --ocp-metadata string          OCP metadata.json file (or directory
containing it) to read AWS infra ID and Region from (Takes precedence over the
flags)
      --profile string               AWS Profile to use for credentials (default
"default")
      --region string                AWS Region
      --target-credentials string    AWS credentials configuration file (default
"/home/user/.aws/credentials")
      --target-infra-id string       AWS infra ID
      --target-ocp-metadata string   OCP metadata.json file (or directory
containing it) to read AWS infra ID and Region from (Takes precedence over the
flags)
      --target-profile string        AWS Profile to use for credentials (default
"default")
      --target-region string         AWS Region

Global Flags:
      --kubeconfig string    absolute path(s) to the kubeconfig file(s)
      --kubecontext string   kubeconfig context to use
```

### How it works

This feature is designed to allow Submariner when the
architecture requires a VPC Peering between two clusters on the same
cloud-provider. VPC Peering between hosts networks allows to deploy submariner
using VXLAN speeding up the throughput thanks to not require IPSec encryption
layer between submariner's gateways.

The new feature includes new subctl set of commands to allow setting up a new
VPC Peering in a similar way as subctl cloud-prepare configures the
inbound/outbound network ports and rules to deploy Submariner.  Those commands
are detailed in [VPC Peering subcomand](#vpc-peering-subcomands) section.

To be able to use this feature, it's mandatory to have the `metadata.json` file
result of installation process of both clusters or specify the required properties
of each cluster using the flags as `--infra-id` , `--region`, `--profile` and
also for the target cluster `--target-infra-id` , `--target-region`, `--target-profile`.

We refer to "source cluster" or "target cluster" to the clusters whose VPCs will
be peered. The "source-cluster" is the one that starts the peering
process, whereas the "target cluster" is the cluster accepting the peering.

### Cloud providing support

As each cloud providers has different implementations of VPC Peering concept,
it's needed to develop specific implementations to each provider. The target
cloud providers are:

* Amazon Web Services (AWS)
* Google Cloud Platform (GCP)
* Microsoft Azure

### Impacted modules

This new feature requires changes in the 'cloud-prepare' and
'submariner-operator' repositories.

The 'cloud-prepare' repository has to support the different ways of Peer VPCs
for each of the target cloud providers in a generic way, including the missing
adaptor methods for the cloud clients.

* AWS
  * aws.AcceptVpcPeeringConnection
  * aws.CreateRoute
  * aws.CreateVpcPeeringConnection
  * aws.DescribeRouteTables
  * aws.DescribeVpcPeeringConnections
  * aws.DeleteRoute
  * aws.DeleteVpcPeeringConnection

* GCP
  * gcp.AddPeering
  * gcp.RemovePeering

* Azure
  * azure.NewVirtualNetworkPeeringsClient
  * azure.VirtualNetworkPeeringsClient.BeginCreateOrUpdate
  * azure.VirtualNetworkPeeringsClient.BeginDelete

For the sake of readability and maintenance, the code can be refactored to follow a
more Object Oriented Programming (OOP) so that the `Cloud` *object* is used
in a similar way regardless whether it is the *source* or the *target*.

```go
source.getVpcID()
target.getVpcID()
source.requestPeering(sourceVpcID, target)
target.acceptPeering(peeringID)
```

Regarding the 'submariner-operator' repository, it will only require adding the logic
to describe the sub-commands detailed before without any other structure or
significant change in the current design. To make easier to use, the new
sub-commands will be able to read the needed parameters from the `metadata.json`
file or using the command line flags.

### Dependencies

This features have impact on `submariner-io/cloud-prepare` and
`submariner-io/submariner-operator`. No extra package dependencies have been
added.

## Owners

* Rubén Romero Montes (ruromero@redhat.com)
* Alejandro Villegas López (avillega@redhat.com)
