# Support Krew way of installing `subctl`

Related Issue:
[Add subctl as kubectl plugin](https://github.com/submariner-io/enhancements/issues/182)

## Background

> Krew is a tool that makes it easy to use kubectl plugins. Krew helps you discover plugins, install and manage them on your machine. It
> is similar to tools like apt, dnf or brew. Today, over [200 kubectl plugins](https://krew.sigs.k8s.io/plugins/) are available on Krew.
>
> A plugin is a standalone executable file, whose name begins with `kubectl-`.
>
> More information can be found on [extending kubectl with plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/)
> page.
>
> On the surface, installing a kubectl plugin seems simple enough – all you need to do is to place an executable in the user’s `PATH`
> prefixed with `kubectl-` – that you may be considering some other alternatives to Krew, such as:
>
> * Having the user manually download the plugin executable and move it to some directory in the PATH
> * Distributing the plugin executable using an OS package manager, like Homebrew (macOS), apt/yum (Linux), or Chocolatey (Windows)
> * Distributing the plugin executable using a language package manager (e.g. npm or go get)
>
> While these approaches are not necessarily unworkable, potential drawbacks to consider include:
>
> * How to get updates to users (in the case of manual installation)
> * How to package a plugin for multiple platforms (macOS, Linux, and Windows)
> * How to ensure your users have the appropriate language package manager (go, npm)
> * How to handle a change to the implementation language (e.g. a move from npm to another package manager)
>
> Krew solves these problems cleanly for all kubectl plugins, since it’s designed specifically to address these shortcomings. With Krew,
> after you write a plugin manifest once your plugin can be installed on all platforms without having to deal with their package managers.

## Summary

Krew is a kubectl plugin that makes lifecycle management of kubectl plugins easy. The idea is to have `subctl` as a kubectl plugin and be
managed by Krew. This makes subctl installation and upgrade easy. This will provide an option to users to install `subctl` other than the
`get.submariner.io` script currently used. This will also provide a base for the efforts to be done to include `subctl` in OpenShift CLI
manager.

## Proposal

The plugin name would be `subm`. The `subctl` binary would be compressed into `.tar.gz` (Krew supports only `.tar.gz` and `.zip`
formats) and its Krew manifest file written. The manifest file would be uploaded to
[Krew's centralized plugin index](https://github.com/kubernetes-sigs/krew-index). Users can then install, uninstall or upgrade `subctl` via
Krew. Upgrading `subctl` via Krew would mean updating `subctl` version in the manifest file and publishing the updated manifest file to Krew
index. This process can be automated using
[krew-release-bot](https://github.com/rajatjindal/krew-release-bot) GHA and [goreleaser](https://goreleaser.com/customization/krew/?h=krew).

## Design

A `.tar.gz` files, along with the current `.tar.xz` files, will be created for each OS and architecture `subctl` is built for, currently.
This would be done by adding a new `make` target in `subctl` repository and running that target during the release process. Another option
is to drop `.xz` extension files and only keep `.gz` extension files. The effect of this change to other projects would need to be
considered, if any. Each file's sha256 checksum would be written to `subctl-checksums.txt`.

The [root command name](https://github.com/submariner-io/subctl/blob/c3485569aa9e684dfc5ecad87668992cf377e1bc/cmd/subctl/root.go#L58) would
need to be made variable based on the argument provided. This would allow running `kubectl-subm` command for the plugin and `subctl` command
for standalone `subctl` binary.

    // rootCmd represents the base command when called without any subcommands.
    var rootCmd = &cobra.Command{
    Use:   filepath.Base(os.Args[0]),
    Short: "An installer for Submariner",
    }

Once `.tar.gz` files are available, Krew plugin manifest file `subm.yaml` for subctl would be written as per
[these instructions](https://krew.sigs.k8s.io/docs/developer-guide/plugin-manifest/#sample-plugin-manifest). The plugin name and Krew
manifest file's name must match. The manifest file will have an entry for each OS and architecture the tar file is generated for and its
sha256. The manifest file would also mention the name and location of the `subctl` binary under the achieved folder and means to run it.

Below is a sample `subm.yaml` manifest file.

    apiVersion: krew.googlecontainertools.github.com/v1alpha2
    kind: Plugin
    metadata:
      name: subm
    spec:
      version: "v0.15.0"
      homepage: https://github.com/submariner-io/subctl
      shortDescription: "Manages Submariner and its services"
      description: |
        CLI to install, uninstall and troubleshoot Submariner on a Kubernetes cluster.
      platforms:
        - selector:
            matchLabels:
              os: linux
              arch: amd64
          uri: https://github.com/submariner-io/releases/releases/download/v0.15.0/subctl-v0.15.0-linux-amd64.tar.gz
          sha256: "eab60750423903167448c4888024336388592e867b95080aaf098bc670268399"
          files:
            - from: "subctl-*/subctl-*"
              to: subctl
          bin: subctl

'files' lists which files should be extracted out from the `from` path under the downloaded archive to the `to` path. 'bin' specifies the
path to the plugin executable among extracted files.

Krew creates a symbolic link named `kubectl-subm`, derived from `metadata.name` field in the manifest file to `subctl` plugin executable
after installation is complete.

## Usage

Once this is done, users would be able to install subctl using Krew:

    kubectl krew install subm
    kubectl krew upgrade subm

and use `subctl` as `kubectl-subm` or `kubectl subm`.

    $ kubectl subm
    An installer for Submariner
    
    Usage:
    kubectl-subm [command]
    
    Available Commands:
    benchmark           Benchmark tests
    cloud               Cloud operations
    completion          Generate the autocompletion script for the specified shell
    deploy-broker       Deploys the broker
    diagnose            Run diagnostic checks on the Submariner deployment and report any issues
    export              Exports a resource to other clusters
    gather              Gather troubleshooting information from a cluster
    help                Help about any command
    join                Connect a cluster to an existing broker
    recover-broker-info Recovers the broker-info.subm file from the installed Broker
    show                Show information about Submariner
    unexport            Stop a resource from being exported to other clusters
    uninstall           Uninstall Submariner and its components
    verify              Run verifications between two clusters
    version             Get version information on subctl
    
    Flags:
    -h, --help   help for kubectl-subm
    
    Use "kubectl-subm [command] --help" for more information about a command.

Krew does not have built-in support for installing specific versions of a plugin. Krew is designed to fetch and install the latest available
version of a plugin from the configured plugin index. However, multiple versions of their plugins can be provided in the plugin index,
allowing users to choose which version to install. This facility will not be provided at the moment but can be enhanced in the future. If
and when it would be supported, the version to be installed could be specified via `kubectl krew install subctl@version`.

## External Dependencies

None.

## User Impact

Users would be able to install `subctl` via Krew.
