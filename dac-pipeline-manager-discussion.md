# DaC Pipeline — Manager Discussion Document

**Prepared by:** Navira  
**Date:** 2026-05-12  
**Purpose:** Summary of design decisions made and open items requiring manager input before pilot launch

---

## Background

The Documentation as Code (DaC) Pipeline automates feature documentation for ExpertFlow's CX
development teams. When a developer opens a feature PR, an AI bot generates a documentation
draft. The developer reviews it. On merge, docs publish to Docusaurus automatically.

This document summarises the key design decision added during architecture review, and two open
gaps that need a decision before the pilot can start.

---

## Key Design Decision: PO Review Step (Post-Merge, Async)

### What was decided

During architecture review, a gap was identified: the pipeline had no mechanism to achieve
"100% explicit doc ownership" — a stated success criterion. The original pipeline also had no
quality gate ensuring customer-facing language was reviewed before going public.

A **Product Owner async review step** was added to the pipeline. Here is how it works:

| Step | Who | What happens |
|------|-----|--------------|
| 1 | Developer | Opens a feature PR — no extra steps |
| 2 | AI Bot | Generates a doc draft automatically |
| 3 | Developer | Reviews the draft (one edit or approval) |
| 4 | Feature merges | Doc publishes to Docusaurus in **draft state** — internal only, not customer-visible |
| 5 | Pipeline (auto) | Creates a GitHub Issue assigned to the PO with a link to the draft doc |
| 6 | Product Owner | Reviews the draft at their own pace, edits if needed, then closes the Issue |
| 7 | Pipeline (auto) | Flips doc from `draft` to `published` — now customer-visible on Docusaurus |

### Why this approach was chosen

The core pipeline principle is: **facilitate teams, don't give them more work.** A PO review
step that blocks the feature merge would violate this — developers would wait on the PO,
creating friction and potential bottlenecks.

The async post-merge model means:
- Feature delivery is **never blocked** by PO availability
- Customers **never see** unreviewed documentation
- The PO has a defined, manageable role without being in the critical path
- Doc ownership is formally recorded (the PO who closes the Issue becomes the named owner)

**SLA for PO review:** Docs should reach `published` state within **5 business days** of merge.
Overdue reviews surface automatically in the weekly Monday coverage report sent to team leads.

---

## Open Gaps Requiring Decisions

Both gaps below must be resolved **before the pilot starts**. Neither requires technical work
from the pipeline team — they are people and configuration decisions.

---

### Gap 1: PO GitHub Handles Per Team

#### What this means

When a feature doc merges, the pipeline automatically creates a GitHub Issue and assigns it to
the PO responsible for that team's docs. To do this, the pipeline needs a config file
(`po-assignments.yml`) that maps each CX team to a PO's GitHub username.

This file does not exist yet. Without it, PO review Issues are created but go unassigned —
nobody is notified, and the review step breaks silently.

#### What is needed

Confirmation of **who the PO is for each CX team**, and their **GitHub username**.

The config file looks like this:

```yaml
teams:
  cx-team-alpha:
    repo: expertflow/cx-alpha
    po: github-username-of-po
  cx-team-beta:
    repo: expertflow/cx-beta
    po: github-username-of-po
```

#### Decision needed from manager

> **Is there one PO across all CX teams, or a different PO per team?**

This changes the approach:

| Scenario | Approach |
|----------|----------|
| One PO for all teams | Single assignment in config — but Issue queue could back up if volume is high. Consider batched weekly digest (one Issue per week) instead of per-feature Issues |
| Different PO per team | Per-team mapping in config — Issues go to the right person automatically, volume is distributed |

#### Action required

Once confirmed, collect each PO's GitHub username and provide to the implementation team.
Populating the config file takes under five minutes once the data is in hand.

---

### Gap 2: Docusaurus Draft-Filter Plugin Selection

#### What this means

Every generated doc has a `status` field in its frontmatter:

```
status: draft    ← set automatically at generation
status: published ← set automatically when PO closes the review Issue
```

By default, Docusaurus publishes every markdown file it finds — it does not read the `status`
field. Without extra configuration, **draft docs would be visible to customers**, defeating
the entire purpose of the two-stage visibility model.

Something must tell Docusaurus: *skip any file where `status` is not `published`*.

#### Two options

**Option A — Custom Docusaurus plugin** *(recommended)*

A small piece of JavaScript (~30 lines) added to `docusaurus.config.js` that reads each
doc's frontmatter before the build and excludes `status: draft` files from the public sidebar
and output.

- All docs stay in one folder — simpler to manage
- Fully controlled by ExpertFlow
- Requires a developer comfortable with Docusaurus configuration JS
- One-time setup, no ongoing maintenance

**Option B — Two separate folders**

Draft docs sync into `docs/features/drafts/` (not in the public sidebar). When the PO
approves, the pipeline moves the file to `docs/features/` — which Docusaurus does publish.

- No custom plugin or JavaScript needed
- More complex pipeline logic (file move operation on approval is harder to make reliable)
- Higher risk of silent failures during the publish step

#### Decision needed from manager

> **Does the implementation team include someone comfortable with Docusaurus configuration JS?**

- If **yes** → Option A. One afternoon of work during `expertflow-docs` setup.
- If **no** → Option B. No JS required, but the pipeline publish step is more complex.

This decision must be made during Step 1 of implementation (setting up the `expertflow-docs`
repository) — it cannot be deferred to later without rework.

---

## Summary: What Manager Input Is Needed

| # | Question | Required by | Blocks |
|---|----------|-------------|--------|
| 1 | Is there one PO for all CX teams, or a PO per team? | Before pilot starts | PO review Issues cannot be assigned without this |
| 2 | Who is the PO for each CX team — and what is their GitHub username? | Before pilot starts | `po-assignments.yml` cannot be populated without this |
| 3 | Does the team have a developer comfortable with Docusaurus config JS? | Before `expertflow-docs` setup | Determines Option A vs Option B for draft filtering |

---

## Reference Documents

All documents are in the [dac-finalization-plan](https://github.com/expertflow/dac-finalization-plan) repository:

| Document | Purpose |
|----------|---------|
| [Product Brief](https://github.com/expertflow/dac-finalization-plan/blob/master/dac-pipeline-product-brief.md) | Goals, scope, success criteria, stakeholders |
| [Architecture Document](https://github.com/expertflow/dac-finalization-plan/blob/master/dac-pipeline-architecture.md) | Full technical design, decisions, data flow, implementation handoff |
