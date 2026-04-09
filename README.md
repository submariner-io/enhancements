# Submariner Enhancement Proposals (SEPs)

<!-- markdownlint-disable line-length -->
[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/4865/badge)](https://bestpractices.coreinfrastructure.org/projects/4865)
[![Periodic](https://github.com/submariner-io/enhancements/workflows/Periodic/badge.svg)](https://github.com/submariner-io/enhancements/actions?query=workflow%3APeriodic)
<!-- markdownlint-enable line-length -->

This repository contains enhancement proposals for submariner-io projects.

## What is a SEP?

A **Submariner Enhancement Proposal (SEP)** is a design document that describes a significant change to any
Submariner project. SEPs provide a consistent format for proposing features, documenting design decisions,
and tracking implementation.

Each SEP is numbered sequentially (`SEP-NNNN`) and lives in the `seps/` directory.

## Submitting a SEP

1. Use [template.md](template.md) as your starting point
2. Assign the next available SEP number
3. Name your file `seps/SEP-NNNN-short-description.md`
4. Set `status: draft` in the frontmatter
5. Submit a PR for community review

## SEP Lifecycle

```text
draft → proposed → accepted → implementing → done
                 ↘ deferred
```

- **draft** — work in progress, not ready for review
- **proposed** — PR open, ready for community review
- **accepted** — merged, implementation can begin
- **implementing** — work in progress
- **done** — fully implemented and verified
- **deferred** — postponed for future consideration

## Template Sections

All SEPs include: **Summary**, **Motivation**, **Design**, **Alternatives Considered**.

Two optional sections support structured implementation:

- **Implementation Plan** — step-by-step guidance with file paths and code.
  Useful for both human contributors following the plan and AI agents executing it.
- **Done When** — verification criteria with executable commands where possible.
  Lets anyone confirm the proposal has been fully implemented.

These sections are optional. Pure design/architecture proposals may not need them.
