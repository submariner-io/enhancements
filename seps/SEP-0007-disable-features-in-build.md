# Disable certain features while building `subctl`

Related Issue:
[Support disabling certain features in builds](https://github.com/submariner-io/enhancements/issues/196)

## Summary

Some builds of `subctl` (e.g. for OCM) don’t need certain sub-commands such as `deploy-broker`, `join` etc. It would be useful to be able
to disable these features so that users don’t even see them in `--help` output.

## Background

Quoting from [Go's official documentation](https://pkg.go.dev/cmd/go#hdr-Build_constraints) to under Go constraints.

> Go provides build constraints to identify files that should be included in the package.
> Build constraints are given by a line comment that begins
>
>     //go:build
>
> Constraints may appear in any kind of source file (not just Go), but they must appear near the top of the file, preceded only by blank
> lines and other line comments. These rules mean that in Go files a build constraint must appear before the package clause.
>
> To distinguish build constraints from package documentation, a build constraint should be followed by a blank line.
>
> A build constraint comment is evaluated as an expression containing build tags combined by ||, &&, and ! operators and parentheses.
> Operators have the same meaning as in Go.

## Proposal

All the sub-commands that are related to deployment would be excluded from builds by adding negated tags via build constraints. The
downstream build of `subctl` would need to be built using the `tags` flag.

## Design

The following `subctl` sub-commands will have build constraints so that they can be excluded from the build based on tags provided at
compile-time.

    cloud
    deploy-broker
    join
    recover-broker-info
    uninstall

The build directive would be `!non-deploy`.

To ignore the above subcommands from the `subctl` build, it would need to be built with `--tags non-deploy` flag.

    go build --tags non-deploy <other arguments>

The built `subctl` binary will not have the above-mentioned subcommands.

Example of `subctl` built by just constraining `deploy-broker`.

    $ ./cmd/bin/subctl
    An installer for Submariner
    
    Usage:
    subctl [command]
    
    Available Commands:
    benchmark           Benchmark tests
    cloud               Cloud operations
    completion          Generate the autocompletion script for the specified shell
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
    -h, --help   help for subctl
    
    Use "subctl [command] --help" for more information about a command.

### Usage

Sub-commands exposed to the users can be controlled.

### Alternatives

None.

## External Dependencies

None.

## User Impact

Users would be shielded from unwanted `subctl` sub-commands.
