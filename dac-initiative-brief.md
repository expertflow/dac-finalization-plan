# Documentation as Code — Initiative Brief

**Organization:** ExpertFlow
**Date:** 2026-05-07
**Status:** Draft — Pending Manager Approval
**Prepared by:** Navira
**Related:** [Problem Statement](problem-statement.md)

---

## Purpose

This document defines the scope, phasing, and success criteria for the Documentation as Code (DaC) initiative at ExpertFlow. It is the next step after alignment on the [problem statement](problem-statement.md) and serves as the basis for solution design approval.

---

## Scope

### What Is in Scope

| Dimension | Definition |
|---|---|
| **Product** | ExpertFlow CX |
| **Documentation type** | Feature and product documentation (customer-facing) |
| **Teams** | All CX development teams (4–5 teams), staggered onboarding |
| **Onboarding approach** | One team leads by 2 weeks as reference implementation; remaining teams follow |

### What Is Out of Scope (for this phase)

| Excluded | Reason |
|---|---|
| Other ExpertFlow products | CX pilot must prove the model first |
| API documentation | Different tooling and workflow — addressed in Phase 2 |
| Internal technical docs | Lower customer-facing urgency — addressed in Phase 3 |
| Release notes | Addressed in Phase 4 (auto-generated from Phase 1 content) |
| Migration of existing Confluence docs | Separate migration project; not a success criterion here |

---

## Phasing

The initiative is structured in four phases, each building on the previous.

| Phase | Documentation Type | Dependency |
|---|---|---|
| **Phase 1** | Feature / product docs (customer-facing) | Pipeline established, pilot team onboarded |
| **Phase 2** | API documentation | Phase 1 stable |
| **Phase 3** | Internal technical docs (architecture, ADRs, guides) | Phase 2 stable |
| **Phase 4** | Release notes (auto-generated from Phase 1 content) | Phase 1 mature |

**Phase 1 is the pilot.** All CX teams participate. One team goes first by two weeks to validate the pipeline before the others onboard.

---

## Success Criteria

Every time a developer finishes a feature and merges their code, we check: did they also update or write the doc for that feature in the same merge? If yes — that counts as covered. If no — that's a gap.

The headline measure is **doc coverage rate** — out of every 10 features merged in a sprint, how many had a doc included? If 9 did, that is 90%.

- **90% by end of pilot** — a small margin is allowed while the process is being established
- **100% by full CX rollout** — no feature ships without a doc, zero exceptions

| Outcome | Measure | Target | By When |
|---|---|---|---|
| Docs ship with features | Doc coverage rate on merged PRs | ≥ 90% | End of pilot sprint |
| Docs publish automatically | Time from merge to live on docs site | ≤ 24 hours | From pipeline go-live |
| Customer impact reduces | Doc-related support tickets for CX | Reduced by 50% | 3 months post-pilot |
| Ownership is explicit | Published docs with a named owner | 100% | End of pilot |

---

## Why Now

- The last two CX releases shipped with documentation that was hastily reviewed after multiple escalations, resulting in inaccurate information reaching end-users.
- When a developer leaves with half-documented features, no PO takes ownership of the gap. Remaining team members attempt to trace and reconcile the affected documentation, but accuracy cannot be independently verified — leaving silent knowledge gaps in the product.
- This reactive pattern is accelerating — as CX grows in features and customers, the cost of each documentation gap compounds. Addressing this now, before scale makes it unmanageable, is the driver for this initiative.

---

## Constraints

- Teams are already on GitHub — solution must stay within the GitHub ecosystem.
- Confluence remains in use for meeting notes and team decisions; only product docs move to Git.
- Changes to the Definition of Done require manager approval before enforcement.
- **Timeline:** *(To be confirmed with manager — by when must the pipeline be operational?)*

---

## Stakeholder Sign-off

| Role | Name | Status |
|---|---|---|
| Prepared by | Navira | ✅ |
| Manager review | | ⏳ Pending |
| Team leads briefed | | ⏳ Pending |

---

**Next step:** Manager approval of this brief → Solution design phase begins
