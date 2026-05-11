# Product Brief: ExpertFlow DaC Pipeline

## Executive Summary

ExpertFlow's CX development teams ship features regularly — but documentation has not kept pace.
Rushed post-release reviews produce inaccurate content. Departing developers leave undocumented
features with no one to verify what's missing. As the product scales, these gaps compound silently.

The Documentation as Code (DaC) Pipeline changes the relationship between code and docs by making
them inseparable. When a developer opens a feature PR, an AI bot analyzes the PR description and
code diff and drafts the corresponding documentation automatically. The developer reviews — not
writes. On merge, documentation publishes to Docusaurus within 24 hours, with no manual steps.

This is not a documentation process. It is a software delivery pipeline that happens to produce
documentation as a natural by-product.

## The Problem

Three failures compound at ExpertFlow today:

**Inaccuracy at release.** Documentation is written after the fact, under pressure, by people
recalling what shipped rather than documenting what they built. Recent releases have contained
factually incorrect documentation as a direct result.

**Knowledge loss at departure.** When developers leave, undocumented features leave with them.
There is currently no mechanism to verify completeness — the gap is invisible until a customer
or support agent hits it.

**No accountability.** Documentation is everyone's responsibility in principle and no one's in
practice. Without a pipeline that connects a feature to its doc, there is no coverage signal,
no owner, and no enforcement point.

The cost: support tickets from customers who can't find accurate answers, and developer time
spent answering questions that documentation should have handled.

## The Solution

The DaC Pipeline embeds documentation into the GitHub workflow CX developers already use:

1. **Developer opens a feature PR** — no extra steps required at this point.
2. **AI bot activates** — analyzes the PR title, description, and code diff; generates a
   documentation draft in the same repository, following ExpertFlow's doc structure.
3. **Developer reviews the draft** — not writes it. One PR comment or approval is the full
   ask. A GitHub status check signals yellow if the draft is unreviewed, but does not block merge.
4. **Feature merges → docs auto-publish** — Docusaurus builds and deploys within 24 hours.
   Documentation coverage is tracked automatically per feature and per team.

The developer's total additional effort: reviewing a draft someone (something) else wrote.

## What Makes This Different

Most DaC approaches treat documentation as a parallel process — a separate task developers
must remember to do. This pipeline inverts the model: **documentation is generated first,
and the developer's job is to correct rather than create.**

The enforcement model is deliberately two-tiered. At the developer level: warn, never block.
A yellow status check signals an unreviewed draft — the PR still merges. Developer autonomy
is preserved; resentment and workarounds are avoided. At the team level: a weekly coverage
report goes automatically to team leads, showing per-team and per-developer doc coverage.
Accountability lives where it actually works — with leads, not with pipeline gates.

Both choices are anchored to a single principle: **facilitate teams, don't give them more work.**

## Who This Serves

**Primary: CX developers** — 4–5 teams across ExpertFlow's CX product. They write code; the
pipeline writes docs. Their role is reviewer, not author. Success for them is merging a
feature and knowing documentation is handled.

**Secondary: Support and customer success teams** — They answer questions daily that accurate
documentation would eliminate. Reduced ticket volume is their success signal.

**Tertiary: Documentation owners** — Each published piece has an explicit owner tracked by
the pipeline. For the first time, coverage is measurable and gaps are visible.

## Success Criteria

| Metric | Target |
|--------|--------|
| Doc coverage rate (features with docs at merge) | ≥ 90% by end of pilot sprint |
| Doc coverage rate at full CX rollout | 100% |
| Time from merge to published doc | < 24 hours |
| Doc-related support ticket reduction | ≥ 50% within 3 months of full rollout |
| Explicit doc ownership assignment | 100% of published materials |

## Scope

**In scope (Phase 1 — this brief):**
- Customer-facing feature and product documentation
- All 4–5 CX development teams (one-team pilot, two-week head start)
- Same-repository documentation alongside feature code
- AI-assisted draft generation from PR context
- Docusaurus publishing pipeline via GitHub Actions

**Explicitly out of scope (Phase 1):**
- API documentation (Phase 2)
- Internal technical documentation (Phase 3)
- Auto-generated release notes (Phase 4)
- Confluence migration

## Vision

If the DaC Pipeline succeeds in Phase 1, ExpertFlow has a proven, repeatable model. Phases 2–4
apply the same pipeline to API documentation, internal technical materials, and release notes —
all flowing through the same GitHub-native infrastructure. By then, "documentation debt" is an
artifact of how the industry used to work, not how ExpertFlow works.

The longer-term state: a developer at ExpertFlow ships a feature and documentation is not a
thought they need to have. It already happened.
