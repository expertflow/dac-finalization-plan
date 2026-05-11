# Team Lead Briefing — DaC Pipeline Initiative

> Purpose: 30-minute briefing for CX team leads. Covers what we are building, why, what it asks of developers, and what we need from leads.

---

## The Problem We Are Solving

- Recent releases shipped with inaccurate documentation — written after the fact, under pressure
- When developers leave, their features leave undocumented with no way to verify the gaps
- Documentation has no owner, no coverage signal, and no enforcement point
- Result: support tickets from customers who cannot find accurate answers; developer time spent on questions docs should handle

---

## What We Are Building

- A GitHub-native Documentation as Code (DaC) pipeline
- Documentation is treated as part of the feature delivery cycle — not an afterthought
- The pipeline is embedded in the GitHub workflow your teams already use — no new tools to learn

---

## How It Works (4 Steps)

1. **Developer opens a feature PR** — nothing extra required at this point
2. **AI bot generates a documentation draft** — analyzes the PR title, description, and code diff; writes the doc automatically in the same repository
3. **Developer reviews the draft** — not writes it; one comment or approval is the full ask; a status check turns yellow if unreviewed but does not block the merge
4. **Feature merges → doc publishes automatically** — Docusaurus builds and deploys within 24 hours; coverage is tracked per feature and per team

**The developer's total additional effort: reviewing a draft the AI already wrote.**

---

## What This Means for Your Developers

- No blank page — the AI drafts; the developer corrects
- No blocked merges — the pipeline signals but never blocks
- No new tools — everything happens inside the GitHub PR they already opened
- Documentation is handled before the feature is forgotten

---

## How Accountability Works

- **Developer level:** A yellow status check appears if the draft is unreviewed — visible, not blocking
- **Team lead level:** A weekly automated report shows doc coverage per team and per developer — you will see who is engaging and who is not
- Accountability lives with leads, not with pipeline gates

---

## Pilot Approach

- One CX team leads the pilot by two weeks before others join
- Pilot team gets hands-on support during setup and first sprint
- Feedback from the pilot shapes the pipeline before full rollout
- We are looking for a willing team with upcoming feature work — this is the conversation we want to have with you

---

## Success Targets

| What | Target |
|------|--------|
| Doc coverage rate by end of pilot sprint | ≥ 90% |
| Doc coverage at full CX rollout | 100% |
| Time from merge to published doc | < 24 hours |
| Reduction in doc-related support tickets | ≥ 50% within 3 months |

---

## What We Need From You

- Awareness: understand what the pipeline asks of your team before it arrives
- Pilot candidacy: consider whether your team is a fit for the two-week lead pilot
- Feedback: flag anything that would create friction for your developers — now, not after rollout

---

## What Happens Next

1. Manager sign-off on product brief (in progress)
2. PRD and technical design (architect involvement)
3. Pilot team confirmed and pipeline built
4. Pilot sprint runs — feedback collected
5. Full CX rollout (staggered, two-week intervals)
