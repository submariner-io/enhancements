# Reorganize to use K8s-inspired concepts/tasks/tutorials/reference sections

## Summary

With our current Getting Started and Operations sections, it is increasingly difficult to scale to add new material without dealing with
significant duplication and ill-fitting organization. Follow the experience of the main Kubernetes documentation by organizing this content
in Concepts, Tasks, Tutorials, and Reference sections.

## Proposal

Reorganize `submariner-website/src/content` to:

```text
.
├── _index.md
├── getting-started
│   ├── _index.md
│   └── known-issues.md # TODO: Combine into index page?
├── concepts
│   ├── _index.md
│   ├── architecture
│   │   ├── _index.md
│   │   ├── broker.md
│   │   ├── gateway-engine.md
│   │   ├── globalnet.md
│   │   ├── route-agent.md
│   │   └── service-discovery.md
│   ├── clusterset.md
│   └── serviceexport-serviceimport.md
├── tasks
│   ├── _index.md
│   ├── infra
│   │   ├── _index.md
│   │   ├── aws.md
│   │   ├── calico.md
│   │   ├── gke.md
│   │   ├── kind.md
│   │   ├── openshift.md
│   │   └── rancher.md
│   ├── deploy
│   │   ├── _index.md
│   │   ├── helm
│   │   │   ├── _index.md
│   │   │   ├── install-charts.md
│   │   │   ├── broker.md
│   │   │   └── join-clusterset.md
│   │   └── subctl
│   │       ├── _index.md
│   │       ├── install-subctl.md
│   │       ├── broker.md
│   │       ├── join-clusterset.md
│   │       └── submariner-with-sd.md
│   ├── verify
│   │   ├── _index.md
│   │   ├── subctl-verify.md
│   │   ├── manual-service-connectivity.md
│   │   └── manual-statefulset.md
│   ├── monitor
│   │   ├── _index.md
│   │   └── enable-prometheus.md
│   └── troubleshoot
│       ├── _index.md
│       ├── connectivity.md
│       └── service-discovery.md
├── tutorials
│   ├── _index.md
│   └── # TODO Nir's WIP "user guide" PR reformatted
├── reference
│   ├── _index.md
│   ├── prometheus-metrics.md
│   └── subctl.md
├── community
│   ├── _index.md
│   ├── code-of-conduct.md
│   ├── contributor-roles.md
│   ├── getting-help.md
│   ├── releases.md
│   └── roadmap.md
├── development
│   ├── _index.md
│   ├── building-testing.md
│   ├── code-review.md
│   ├── release-process.md
│   ├── security.md
│   ├── shipyard
│   │   ├── _index.md
│   │   ├── first-time.md
│   │   └── advanced.md
│   └── website
│       ├── _index.md
│       └── style_guide.md
├── other-resources
│   └── _index.md
└── security.txt
```

For reference, the current website layout is:

```text
.
├── community
│   ├── code-of-conduct
│   │   └── _index.en.md
│   ├── contributor-roles
│   │   └── _index.en.md
│   ├── getting-help
│   │   └── _index.en.md
│   ├── _index.en.md
│   ├── releases
│   │   └── _index.en.md
│   └── roadmap
│       └── _index.en.md
├── development
│   ├── building-testing
│   │   └── _index.en.md
│   ├── code-review
│   │   └── _index.en.md
│   ├── _index.en.md
│   ├── release-process
│   │   └── _index.en.md
│   ├── security
│   │   └── _index.en.md
│   ├── shipyard
│   │   ├── advanced
│   │   │   └── _index.en.md
│   │   ├── first-time
│   │   │   └── _index.en.md
│   │   └── _index.en.md
│   └── website
│       ├── _index.en.md
│       └── style_guide.md
├── getting-started
│   ├── architecture
│   │   ├── broker
│   │   │   └── _index.en.md
│   │   ├── gateway-engine
│   │   │   └── _index.en.md
│   │   ├── globalnet
│   │   │   └── _index.en.md
│   │   ├── _index.en.md
│   │   ├── route-agent
│   │   │   └── _index.en.md
│   │   └── service-discovery
│   │       └── _index.en.md
│   ├── _index.en.md
│   └── quickstart
│       ├── _index.en.md
│       ├── kind
│       │   └── _index.md
│       ├── managed-kubernetes
│       │   ├── gke
│       │   │   └── _index.md
│       │   ├── _index.en.md
│       │   └── rancher
│       │       └── _index.md
│       └── openshift
│           ├── aws
│           │   └── _index.md
│           ├── globalnet
│           │   └── _index.md
│           ├── _index.md
│           └── vsphere-aws
│               └── _index.md
├── _index.en.md
├── operations
│   ├── deployment
│   │   ├── calico
│   │   │   └── _index.en.md
│   │   ├── helm
│   │   │   └── _index.en.md
│   │   ├── _index.en.md
│   │   └── subctl
│   │       └── _index.en.md
│   ├── _index.en.md
│   ├── known-issues
│   │   └── _index.en.md
│   ├── monitoring
│   │   └── _index.en.md
│   └── troubleshooting
│       └── _index.en.md
├── other-resources
│   └── _index.en.md
└── security.txt
```
