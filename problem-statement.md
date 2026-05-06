# Problem Statement: Documentation as Code

**Organization:** ExpertFlow
**Date:** 2026-05-06
**Status:** Problem Definition Phase

---

## Core Problem Statement

> ExpertFlow teams produce documentation as a release-gate ritual rather than as a living artifact of the development process. Because documentation is structurally decoupled from the code workflow — written separately, stored separately, reviewed separately, and published manually — it systematically arrives too late, too incomplete, and under conditions where no one has capacity to improve it. The result is a persistent gap between what the product does and what is documented.

---

## Current State

Teams (developers and Product Owners) author documentation in Confluence and internal team wikis throughout the development cycle. Documentation is not co-located with code, has no coupling to Pull Requests or version control, and carries no enforcement mechanism within the Definition of Done. At release time, documentation is compiled from scattered sources, reviewed under time pressure by people simultaneously managing the release, and published manually — producing output that is incomplete, inconsistent, and frequently misaligned with what was actually shipped.

---

## Root Causes

**1. Wrong timing**
Documentation is written after development, from memory, at the moment of lowest available context and highest team workload. The developer who knows the most about a feature knows it *during* development, not at release.

**2. Wrong location**
Confluence is a wiki — a destination for collaborative notes. It has no awareness of the code repository. When code changes, Confluence does not know. There is no mechanism to detect that a feature changed and its documentation is now stale.

**3. No enforcement**
Documentation has no binding relationship to the delivery workflow. No PR is blocked for missing docs. No pipeline fails for stale docs. No release gate checks doc completeness. Without enforcement, documentation competes — and loses — against work that does have enforcement.

**4. No ownership model**
Responsibility for documentation sits ambiguously between the developer (who understands the implementation) and the Product Owner (who understands the intent). Without a clear owner per PR, ownership defaults to nobody.

---

## Business Impact

| Area | Consequence |
|---|---|
| Customer experience | Incomplete docs shipped with features create support overhead and erode trust |
| Team velocity | New developers spend significant time reconstructing knowledge that should be written down |
| Release quality | Docs written from memory introduce inaccuracies that reach customers |
| Knowledge retention | When a developer or PO leaves, undocumented feature context leaves with them |
| Review quality | Doc reviews at release are rubber stamps — reviewers lack capacity to meaningfully engage |

---

## What Documentation as Code Addresses

Documentation as Code is an industry practice that resolves these root causes by treating documentation with the same discipline as code:

- Documentation is authored in Markdown, stored in the same Git repository as the code it describes
- Documentation changes are submitted in the same Pull Request as the code they describe
- Documentation is reviewed by the same reviewers at PR time, not at release time
- Documentation is automatically published via CI/CD pipeline on merge — no manual step
- A PR that changes user-facing behavior without a corresponding doc update does not merge

This shifts documentation from a release ritual to a continuous, enforced, automated practice.

---

## What This Problem Statement Does Not Cover

This document defines the problem. Pipeline design, tooling selection, migration strategy, and rollout planning are subsequent phases, contingent on alignment on this problem definition.

---

**Prepared by:** Nabiha
**Next step:** Alignment on problem statement → solution design phase
