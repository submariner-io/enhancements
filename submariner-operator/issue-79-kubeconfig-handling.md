## Epic Description

The various `subctl` commands support a mixture of `KUBECONFIG` environment variables, `--kubeconfig` flags, `--kubecontext` flags,
kubeconfig files etc.

We should make this consistent, ideally using `clientcmd`’s tools to handle kubeconfig.
[Operator PR #23](https://github.com/skitt/submariner-operator/pull/23) has an old attempt at this,
using [`BindOverrideFlags()`](https://pkg.go.dev/k8s.io/client-go@v0.23.5/tools/clientcmd#BindOverrideFlags) to provide access
to all the features supported by `clientcmd` (see `kubectl options`).

On top of that, we should align our flags with `kubectl`’s;
beyond helping users already familiar with `kubectl` flags,
this will make it easier to turn `subctl` into a `kubectl` plugin,
should we choose to do so.
This would involve turning `--kubecontext` into `--context`;
`--kubecontext` would be preserved as a deprecated alias for `--context` for one release,
and removed in the following release.

## Acceptance Criteria

For commands using a single context, _all_ the following should work,
with kubeconfigs containing one or more clusters and/or contexts:

```bash
# Use the default context in the kubeconfig
KUBECONFIG=/path/to/kubeconfig subctl foo
subctl foo --kubeconfig /path/to/kubeconfig
subctl --kubeconfig /path/to/kubeconfig foo

# Use the specified context from the kubeconfig
KUBECONFIG=/path/to/kubeconfig subctl foo --context bar
subctl foo --kubeconfig /path/to/kubeconfig --context bar
subctl --kubeconfig /path/to/kubeconfig foo --context bar
subctl --kubeconfig /path/to/kubeconfig --context bar foo
```

(and other variations on option order).

For commands using multiple contexts,
there should be one “main” set of flags, with no prefix,
and another set with a prefix describing the purpose of the context:

```bash
KUBECONFIG=/path/to/kubeconfig subctl benchmark latency --context foo --tocontext bar
subctl benchmark latency --kubeconfig /path/to/kubeconfig --context foo --tocontext bar
```

`--kubeconfig` is preserved as-is, since users can use a single file combining all their contexts.
However such commands should also support separate kubeconfigs, to allow using kubeconfigs which
contain conflicting configuration (_e.g._ kubeconfigs using the same context name and user name,
as produced by the OpenShift installer):

```bash
subctl benchmark latency --kubeconfig /path/to/fromconfig --toconfig /path/to/toconfig
```

These can be combined:

```bash
subctl benchmark latency --kubeconfig /path/to/fromconfig --context foo --toconfig /path/to/toconfig --tocontext bar
```

If different kubeconfigs are specified, each context will only be loaded from the corresponding
kubeconfig.

(When presented with conflicting configurations, `kubectl` only uses the first relevant context,
the other contexts are inaccessible unless the conflicts are resolved.
For example, with OpenShift `kubeconfig` files and their `admin` context, only the `admin` context
in the first file listed in `$KUBECONFIG` can be used.)

For commands potentially using _all_ accessible contexts (`subctl gather`), the existing behaviour
should be preserved:

```bash
# Use all accessible contexts
KUBECONFIG=/path/to/kubeconfig subctl gather
subctl gather --kubeconfig /path/to/kubeconfig
subctl gather /path/to/kubeconfig

# Use one named context
KUBECONFIG=/path/to/kubeconfig subctl gather --context context1
subctl gather --kubeconfig /path/to/kubeconfig --context context1
subctl gather /path/to/kubeconfig --context context1

# Use multiple named contexts
KUBECONFIG=/path/to/kubeconfig subctl gather --contexts context1,context2
subctl gather --kubeconfig /path/to/kubeconfig --contexts context1,context2
subctl gather /path/to/kubeconfig --contexts context1,context2
```

`clientcmd` _doesn’t_ handle `--kubeconfig` itself, that needs to be handled by `subctl`.

This enhancement should also include a test suite covering all the different possibilities.
As far as possible, the options present in the previous release should keep their existing behaviour,
and be marked for deprecation so that they can be removed in a future release.

This would subsume [`--kubecontexts` support in `subctl diagnose`](https://github.com/submariner-io/submariner-operator/issues/1327).

## Work Items

* Create a test suite for the various `--kubeconfig` and `--kubecontext` variants
* Delegate flag setup for `--kubecontext` and related flags to `k8s.io/client-go/tools/clientcmd`
* Integrate clientcmd handling into `restconfig.Producer` (if it still exists)
* Where appropriate, add `--kube` variants for command which need multiple contexts

It will probably be necessary to isolate the `clientcmd` data structures in each command using them,
to ensure that uninitialised data structures can’t be accidentally used.
See [`kubectl apply`](https://github.com/kubernetes/kubectl/blob/master/pkg/cmd/apply/apply.go) for
an example of data structure use to isolate flags and options.

This work will flip the way `clientcmd` settings are handled.
Each command using `clientcmd` will need to maintain a `clientcmd.ConfigOverrides` instance as part
of its state (_e.g._ in a structure describing its settings), and “hook up” the various flags to
fields in that structure: `--kubeconfig` maps to `clientcmd.DefaultClientConfig.ExplicitPath`,
the remaining flags are set up with `clientcmd.BindOverrideFlags` using an appropriate prefix
(empty in most cases, so the flags end up being `--context` etc.).
The plural form, `--contexts`, will need special handling.

The following illustrates the initial setup:

```go
// addKubeContextFlag adds a "kubeconfig" flag and a single "context" flag that can be used once and only once
func addKubeContextFlag(cmd *cobra.Command) {
    loadingRules := clientcmd.NewDefaultClientConfigLoadingRules()
    loadingRules.DefaultClientConfig = &clientcmd.DefaultClientConfig
    overrides := clientcmd.ConfigOverrides{}
    kflags := clientcmd.RecommendedConfigOverrideFlags("")
    addKubeConfigFlag(cmd, &loadingRules.ExplicitPath)
    clientcmd.BindOverrideFlags(&overrides, cmd.PersistentFlags(), kflags)
    clientConfig = clientcmd.NewNonInteractiveDeferredLoadingClientConfig(loadingRules, &overrides)
}
```
