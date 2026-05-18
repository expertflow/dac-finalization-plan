# Problem Statement: Documentation as Code

**Organization:** ExpertFlow  
**Date:** 2026-05-06  
**Status:** Problem Definition Phase  
**Prepared by:** Navira  

---

## The Gap

> At ExpertFlow, code and documentation live in separate worlds. Code is versioned, reviewed, and deployed as part of a single, automated pipeline. Documentation is written after the fact, stored in a wiki, and published by hand. The result is predictable: documentation does not describe what the product currently does. It describes what someone remembers about what it once did.

This is not a documentation quality problem. It is a **workflow design problem**.

---

## Current State

### How Features Become Docs Today

| Stage | What Happens | Where It Lives |
|-------|-------------|----------------|
| 1. PO defines feature | Intent captured in Jira / Confluence | Confluence |
| 2. Developer implements | Code written, reviewed, merged | GitHub |
| 3. Testing surfaces gaps | Limitations discovered, scope shifts | Jira comments |
| 4. Release approaches | Docs compiled from scattered sources | Confluence (new pages) |
| 5. Release gate review | Docs reviewed under time pressure | Confluence (same pages) |
| 6. Publication | Manual copy-paste or page updates | Confluence (public space) |

At no point does the documentation workflow intersect the code workflow. A developer can merge code without any documentation action. A PR can be approved without any doc review. The two systems are structurally decoupled.

### Consequences of Decoupling

- **Version drift:** The code on `main` advances; the documentation in Confluence does not know. There is no signal that a feature has changed and its documentation is now stale.
- **Context loss:** Documentation is written from memory, often weeks after implementation, by people who have since moved to new work. Edge cases, constraints, and behavior discovered during development are lost.
- **Ownership void:** Responsibility sits between the developer (who understands the implementation) and the PO (who understands the intent). Without a clear owner at the point of change, ownership defaults to nobody.
- **Enforcement absence:** No PR is blocked for missing docs. No pipeline fails for stale docs. No release gate checks doc completeness. Work that has no enforcement competes — and loses — against work that does.

---

## Root Causes

### 1. Temporal Decoupling

Documentation is produced *after* development, at the moment of lowest available context and highest team workload. The developer who knows the feature best knows it *during* implementation, not at release. By the time documentation is requested, that developer has mentally moved on.

### 2. Spatial Decoupling

Confluence is a collaborative wiki, not a versioned artifact store. It has no awareness of the code repository, the PR that introduced a change, or the branch where that change lives. When code changes, Confluence does not know. There is no diff, no blame, no history that correlates a documentation change with a code change.

### 3. Workflow Decoupling

The software delivery pipeline enforces tests, code review, and build verification. Documentation has no equivalent gate. It is not part of the Definition of Done. It is not checked in CI. It is not required for merge. Without embedding documentation into the same workflow that produces the code, it will always be treated as optional.

### 4. Ownership Ambiguity

There is no single point of accountability for documentation at the PR level. The developer knows the implementation but may not have the product lens. The PO has the product lens but lacks implementation detail. Without a mechanism that requires both to review at the moment of change, each assumes the other will handle it — and neither does.

---

## Business Impact

| Area | Consequence | Evidence |
|------|-------------|----------|
| **Customer experience** | Incomplete or inaccurate docs shipped with features create support overhead and erode trust | Recent CX releases shipped with documentation that required post-release corrections after customer escalations |
| **Team velocity** | New developers spend time reconstructing knowledge that should be written down | *(To be quantified — onboarding time for developers without doc coverage)* |
| **Release quality** | Docs written from memory introduce inaccuracies that reach customers | Factual errors in shipped documentation traced to post-hoc authoring |
| **Knowledge retention** | When a developer or PO leaves, undocumented feature context leaves with them | Undocumented features with no remaining subject-matter expert |
| **Review quality** | Doc reviews at release are rubber stamps — reviewers lack capacity to meaningfully engage | Release documentation reviewed under time pressure by staff simultaneously managing the release |

---

## Desired State

The goal is not "better documentation." The goal is a **single workflow** where documentation is as integral to software delivery as tests and code review.

In the desired state:

- Documentation lives in the same repository as the code it describes, versioned alongside it.
- A change to user-facing behavior cannot merge without a corresponding documentation update.
- Reviews happen at PR time, with full context, by the people best positioned to judge accuracy — not weeks later at release.
- Documentation publishes automatically on merge. No manual publication step.
- Every published document has a named owner. Coverage is measurable; gaps are visible.

---

## What This Document Does Not Cover

This document defines the problem. Pipeline design, tooling selection, migration strategy, and rollout planning are subsequent phases, contingent on alignment on this problem definition.

---

**Next step:** Alignment on problem statement → solution design phase
