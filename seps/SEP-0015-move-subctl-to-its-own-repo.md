# Move `subctl` to its own repository

## Summary

Currently, `subctl` code lives in the `submariner-operator` repository. Both `subctl` and the Operator require different sets of
dependencies which might conflict with improving either or both. For example, `subctl` can't be `go install`-ed currently because the
Operator dependencies don't allow it. It also makes Operator upgrades difficult.

## Proposal

The proposal is to move `subctl` out of the `submariner-operator` repository to its own repository. Everything specific to `subctl` will be
moved to the new `subctl` repository. The `subctl` project will depend on the Operator (_e.g._ for the CRDs, RBAC etc. which are defined in
the Operator). The `submariner-operator` repository will only contain artifacts which the Operator framework generates and/or needs.

### Pros

* Operator framework upgrades would be simpler.
* It's logically correct to separate the Operator and the library as they are 2 different deliverables and provide different ways to
  install Submariner.
* Testing criteria are different. We don't need `subctl`-related GHAs for Operator and vice versa.
* Development approach is different. One is a CLI and another is a framework.
* Integration into `submariner-addon` might be simplified.

### Cons

* A new repository to maintain.
* Splitting up the Operator and `subctl` will require care when implementing changes in either which affect the other.

## Design details

A new GitHub repository `subctl` will be created under the `submariner-io` organisation. The `submariner-operator` repository content will
be copied to the new `subctl` repository, by pushing the git repository as-is, so that the full history is preserved. Non-`subctl`-related
content will then be removed from the `subctl` repository. This includes cobra and library portion of `subctl`. Non-Operator-related
content will be removed from the `submariner-operator`repository. Once this is complete, there should be no duplicate code content in the
two repositories, other than the build framework. The `subctl` repository will be set up with the same permissions, branch setup, and code
ownership as the `submariner-operator` repository. The `subctl` project will depend on the Operator (_e.g._ for the CRDs, RBAC etc. which
are defined in the Operator).

## Work items

### On `subctl` repository

* Create a new `subctl` repository and copy all the contents from `submariner-operator` repository.
* Delete Operator specific code. In particular, `bundle`, `config`, `controllers`, `deploy`, `packagemanifests`, `package/*-operator*`,
  `scripts/operator*`, `bundle.Dockerfile` directories/files.
* Change module name in `go.mod` to `github.com/submariner-io/subctl` and add a dependency on `submariner-operator`.
* Delete `e2e-all-k8s.yml` CI jobs.

### On `submariner-operator` repository

* Delete the `subctl`-specific code.
* Delete `subctl.yml`, `cross.yml` and `release_subctl_on_push.yml` CI jobs.

### On `get.submariner.io` repository

* Change URLs to download `subctl` binary from new `subctl` repository.

### On `submariner-website` repository

* Point links to new `subctl` repository wherever applicable.

### Release process

* Add a `subctl` phase in release automation after `submariner-operator` phase.

### External dependencies

* Add `subctl` dependency in `submariner-addon`.

## User Impact

No user impact is anticipated.
