## Epic Description

The operator was built using an old version of the
[Operator SDK](https://sdk.operatorframework.io/docs/building-operators/golang/quickstart/).
This causes a number of problems:

* our code (in particular, metrics setup) relies on an old version of the Operator framework,
  which is incompatible with current versions of the Kubernetes libraries
* we’re missing out on features supported by newer versions of the SDK (such as healthchecks)

We need to upgrade our operator to use the current Operator SDK, following
[the migration guide](https://sdk.operatorframework.io/docs/building-operators/golang/migration/).

This work will be easier once `subctl` is in its own repository; see
[Epic: Move subctl to its own repo](https://github.com/submariner-io/enhancements/issues/89).
Without this work, a non-negligible amount of time would probably go into avoiding `subctl`
breakage while reworking the operator itself.

## Acceptance Criteria

All our end-to-end tests pass with the new operator.

## Definition of Done (Checklist)
<!-- Make sure to check all relevant items before end of the release: -->

* [ ] Code complete
* [ ] The acceptance criteria met
* [ ] Unit/e2e test added & pass
* [ ] CI jobs pass
* [ ] Deployed using cloud-prepare+subctl
* [ ] Run subctl verify, diagnose and gather
* [ ] Uninstall
* [ ] Troubleshooting (gather/diagnose) added
* [ ] Documentation added
* [ ] Release notes added

## Work Items

* [ ] Create a new operator using the Operator SDK, preserving our existing group and identifiers
* [ ] Integrate the existing controllers into the new framework.
* [ ] Make sure the setup work done in our code is replicated in the new framework:
  * [ ] Remove code which takes care of features supported elsewhere (metrics)
  * [ ] Determine how to handle unorthodox features in our code (CRD setup; see the addon for inspiration)

The following considerations should be at least borne in mind while working on this, if not addressed:

* permissions should be minimal, and if possible, cluster roles should be avoided entirely; see
  [Epic: descope the Submariner operator](https://github.com/submariner-io/enhancements/issues/75) and
  [the descoping plan](https://hackmd.io/wVfLKpxtSN-P0n07Kx4J8Q)
* [RBAC should be auto-generated](https://github.com/submariner-io/submariner-operator/issues/1105) based
  on metadata in the source code, rather than manually maintained in YAML files
