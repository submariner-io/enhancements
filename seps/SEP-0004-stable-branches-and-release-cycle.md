# Producing and supporting stable releases

## Summary

Today the Submariner project releases one minor version a month without capacity
for stable branches or bugfix releases. In the following proposal we define a plan
to make this possible starting with 0.8.

## Proposal

### Alignment of branch -> image releases

We currently produce `<image>:devel` for every master push, along with `<image>:x.y.z` for
the releases.

Changing the `master` branch name to `devel` would make this homogeneous so our image releases
are simply `<image>:<branch-or-tag>`.

### Creating one stable branch for every minor release

Approaching the first release candidate `rc0` of every minor or major version, a new stable
branch will be forked off `devel` branch in the form of `release-x.y` by the release scripts,
and a patch or PR pinning the shipyard-base-image to `release-x.y` will be applied by the
automated process.

Any following changes related to pinning go modules in submariner, admiral, lighthouse, etc...
will be done in the `release-x.y` branch directly (no cherry picks).

Once the branch exists, any patches which should be part of the release will need to be
merged in `devel`and backported to `release-x.y`.

### Maintaining a stable branch

#### Latest stable branch (N)

The only patches that should be submitted to the latest stable branch are
bugfixes and not features, unless for exceptions previously agreed on the
community meeting.

Any patches that are necessary for fixing bugs in a stable branch will
need to be merged first into `devel`, and then cherry-picked to the stable
branch.

`release-0.9` <-- `devel` <-- `bug-fix patch`

*No-change version bumps* will be done if a base image has been updated
due to a security bugfix, our code would remain the same, but our versions
would be bumped to pickup the new images.

#### Previous stable branch (N-1)

The only patches that should be submitted to the previous stable branch (N-1)
are critical bugfixes, being the definition of critical bugfix: security
related fixes, and CVEs.

The patches necessary for the critical bugfixes should be cherry-picked from
previous  versions, though sometimes they could need specific adaptations
to the branch.

`release-0.8`  <-- `release-0.9` <-- `devel` <-- `bug-fix patch`

*No-change version bumps* will be necessary if a base image has been updated
due to a security bugfix, our code would remain the same, but our versions
would be bumped to pickup the new images.

#### Releasing a stable branch

When a patch is merged to a stable branch the images for such branch will be updated, for
example `quay.io/submariner/submariner:release-x.y`.

This image allows for testing a set of bugfix patches, sometimes multiple patches are expected
to be required for one fix. Once the release branch is ready we should create a new x.y.**z**
patch release via the release repository.

### How long is a stable branch maintained

The current stable branch (N) will be maintained for normal bugfixes until
the next stable version is released. The previous stable branch (N-1) will be
maintained only for critical bugfixes, however this can change in the future if
we introduce the concept of long term support releases.

### Reducing the cadence of minor releases

Based on the previous ideas, the existing minor version cadence doesn't seem realistic
for maintaining stable branches. We don't want "fake" stable branches where patches
are never backported.

With the new cadence, after each sprint we would produce a milestone release,
for example

* `0.9-milestone-1-rc0`, ... , `0.9-milestone-1`
* `0.9-milestone-2-rc0`, ... , `0.9-milestone-2`
* `0.9-milestone-3-rc0`, ... , `0.9-milestone-3`
* `0.9-milestone-4-rc0`, ... , `0.9-milestone-4`

In this case we won't do pinning of shipyard base image, but we will release and update
the admiral & shipyard `go.mod` git references, without pinning shipyard.

After a number of milestones instead we would follow the usual pattern for a release:

* `0.9.0-rc0` along with its stable branch `release-0.9` as described
  previously, ..., `0.9.0`

The actual cadence and release times is TBD by the community, it could be aligned with
k8s for example, slightly after a release of k8s we could adapt to the new changes
in k8s API and produce a final release.

## Work items

* [ ] all repositories: produce `<image>:<branch>` images on push
* [ ] releases repo support for forking repositories approaching a stable release.
* [ ] all necessary changes in the releases repository
* [ ] all necessary changes in CI scripts.
