---
sep: 31
title: "Modernize Enhancement Proposals"
status: draft
authors: [dfarrell07]
created: 2026-04-09
components: [enhancements]
repos: [submariner-io/enhancements]
---

# Modernize Enhancement Proposals

## Summary

Adopt KEP-style conventions for Submariner Enhancement Proposals (SEPs): numbered proposals,
YAML frontmatter, flat directory structure, and optional implementation plan sections that
work for both human and agent-driven development.

## Motivation

The enhancements repo has served the project well with 30 proposals across 5 component
directories. However, it predates the era of AI-assisted development and lacks conventions
that modern projects use:

- **No numbering** — proposals are named ad-hoc (`issue-182-subctl-a-kubectl-plugin.md`)
  with no canonical identifier for cross-referencing
- **No frontmatter** — status, authorship, and component info live in inconsistent inline text
  or don't exist at all
- **Component-based directories** — proposals that span multiple repos
  (e.g., globalnet touches submariner + operator) don't fit cleanly
- **No implementation guidance** — proposals describe *what* to build but not *how*,
  leaving implementation up to tribal knowledge
- **No completion criteria** — no way to verify a proposal has been fully implemented

Kubernetes KEPs, Python PEPs, and Rust RFCs all solved these problems years ago.
The Agent Skills open standard (adopted by 33+ tools) means structured proposals
can now be consumed by any contributor's AI tool, making this a good time to modernize.

## Design

### SEP Numbering

Every proposal gets a sequential number: `SEP-NNNN`. This provides a stable identifier
for discussion ("see SEP-0013") regardless of renames or moves.

Existing 30 proposals are retroactively numbered SEP-0001 through SEP-0030 based on
alphabetical order within component directories (lighthouse, releases, subctl,
submariner-operator, submariner, then root).

### YAML Frontmatter

Every SEP includes machine-parseable frontmatter:

```yaml
---
sep: NNNN
title: "Human-readable title"
status: draft | proposed | accepted | implementing | done | deferred
authors: [github-handles]
created: YYYY-MM-DD
components: [submariner, lighthouse, submariner-operator, subctl, shipyard]
repos: [submariner-io/repo-name]
---
```

### Flat Directory Structure

Replace component directories with a single `seps/` directory:

```text
enhancements/
├── seps/
│   ├── SEP-0001-aggregate-service-import.md
│   ├── SEP-0002-mcs-compliance.md
│   ├── ...
│   └── SEP-0031-modernize-enhancements.md
├── template.md
└── README.md
```

The `components` field in frontmatter replaces directory-based organization.
Proposals spanning multiple components no longer need to pick one directory.

### Updated Template

The template adds two optional sections beyond the existing structure:

**Implementation Plan** — step-by-step guidance with file paths and code blocks.
Useful for both human contributors following the plan and agents executing it.
Not required for pure design/architecture proposals.

**Done When** — verification criteria, ideally with executable commands.
Lets anyone (human or agent) confirm the proposal has been fully implemented.

### Alternatives Considered

**Separate directories for "human proposals" vs "agent plans"** — rejected because
the distinction is artificial. A well-structured proposal is useful to both humans
and agents. One format, one numbering sequence.

**Agent Skills format** — rejected for proposals. Skills are for recurring operations
(release management, CVE fixes). Proposals are one-time designs. Different lifecycle.

**New repo** — rejected. The enhancements repo already has CI, CODEOWNERS, and
community process. Lower friction to evolve it.

## Implementation Plan

### Step 1: Create `seps/` directory and move existing proposals

Move and rename all 30 existing proposals into `seps/` with SEP numbering.

Numbering assignment (alphabetical within component groups):

| SEP | Current Path | New Filename |
| --- | ------------ | ------------ |
| 0001 | lighthouse/aggregate-service-import.md | SEP-0001-aggregate-service-import.md |
| 0002 | lighthouse/mcs-compliance.md | SEP-0002-mcs-compliance.md |
| 0003 | lighthouse/srv-records.md | SEP-0003-srv-records.md |
| 0004 | releases/stable-branches-and-release-cycle.md | SEP-0004-stable-branches-and-release-cycle.md |
| 0005 | subctl/issue-182-subctl-a-kubectl-plugin.md | SEP-0005-subctl-kubectl-plugin.md |
| 0006 | subctl/issue-183-support-upgrading-submariner.md | SEP-0006-support-upgrading-submariner.md |
| 0007 | subctl/issue-196-disbale-features-in-build.md | SEP-0007-disable-features-in-build.md |
| 0008 | subctl/issue-798-show-versions-from-logs.md | SEP-0008-show-versions-from-logs.md |
| 0009 | subctl/issue-89-recovering-broker-info.md | SEP-0009-recovering-broker-info.md |
| 0010 | submariner-operator/diagnose-refactor-ocm.md | SEP-0010-diagnose-refactor-ocm.md |
| 0011 | submariner-operator/issue-149-node-placement.md | SEP-0011-node-placement.md |
| 0012 | submariner-operator/issue-359-subctl-info-status.md | SEP-0012-subctl-info-status.md |
| 0013 | submariner-operator/issue-79-kubeconfig-handling.md | SEP-0013-kubeconfig-handling.md |
| 0014 | submariner-operator/issue-81-operator-rework.md | SEP-0014-operator-rework.md |
| 0015 | submariner-operator/move-subctl-to-its-own-repo.md | SEP-0015-move-subctl-to-its-own-repo.md |
| 0016 | submariner-operator/submariner-metrics-redesign.md | SEP-0016-metrics-redesign.md |
| 0017 | submariner/cable-driver-policy.md | SEP-0017-cable-driver-policy.md |
| 0018 | submariner/disable-intra-cluster-connectivity.md | SEP-0018-disable-intra-cluster-connectivity.md |
| 0019 | submariner/globalnet-enhancement2-0.md | SEP-0019-globalnet-2.0.md |
| 0020 | submariner/globalnet-ovn.md | SEP-0020-globalnet-ovn.md |
| 0021 | submariner/ipsec-certificate-support.md | SEP-0021-ipsec-certificate-support.md |
| 0022 | submariner/ipsec-cert-issuer.md | SEP-0022-ipsec-cert-issuer.md |
| 0023 | submariner/IPSec-transport-mode.md | SEP-0023-ipsec-transport-mode.md |
| 0024 | submariner/IPV6-datapath.md | SEP-0024-ipv6-datapath.md |
| 0025 | submariner/IPV6-OVN.md | SEP-0025-ipv6-ovn.md |
| 0026 | submariner/multiple-active-gateways.md | SEP-0026-multiple-active-gateways.md |
| 0027 | submariner/OVN-Interconnect.md | SEP-0027-ovn-interconnect.md |
| 0028 | submariner/pluggable-network-plugin-and-ovn.md | SEP-0028-pluggable-network-plugin-and-ovn.md |
| 0029 | submariner/vxlan-tunnel.md | SEP-0029-vxlan-tunnel.md |
| 0030 | issue-40-built-in-benchmarking-tool.md | SEP-0030-built-in-benchmarking-tool.md |

```bash
cd ~/go/src/submariner-io/enhancements
mkdir -p seps

# Move and rename (preserving git history with git mv)
git mv lighthouse/aggregate-service-import.md seps/SEP-0001-aggregate-service-import.md
# ... repeat for all 30 proposals
```

### Step 2: Add frontmatter to existing proposals

Add minimal frontmatter to each moved proposal. Status for all existing proposals
is `done` or `deferred` (they're all historical).

### Step 3: Update template

Update `template.md` with frontmatter block and optional Implementation Plan / Done When sections.

### Step 4: Rewrite README

Explain:

- What SEPs are and how numbering works
- The template and its sections
- How to submit a new SEP
- Status lifecycle: draft → proposed → accepted → implementing → done | deferred

### Step 5: Remove empty component directories

```bash
rmdir lighthouse releases subctl submariner submariner-operator
```

### Step 6: Update CI (optional, follow-up)

Add frontmatter schema validation to linting workflow.

## Done When

```bash
cd ~/go/src/submariner-io/enhancements

# All proposals moved to seps/
ls seps/SEP-*.md | wc -l
# Expected: 31 (30 existing + this one)

# No proposals remain in old component dirs
find lighthouse releases subctl submariner submariner-operator -name "*.md" 2>/dev/null | wc -l
# Expected: 0

# Frontmatter present in all SEPs
for f in seps/SEP-*.md; do head -1 "$f" | grep -q "^---" || echo "Missing frontmatter: $f"; done

# README references SEP process
grep -q "SEP" README.md
```

- [ ] All 30 existing proposals moved and numbered
- [ ] Frontmatter added to all proposals
- [ ] Template updated
- [ ] README rewritten
- [ ] CI still passes (markdown lint, link check)

## Work Items

- [ ] PR with repo restructuring + this SEP
- [ ] Community meeting presentation
- [ ] First feature SEP using new format (validates the template)
