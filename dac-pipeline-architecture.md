---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7]
inputDocuments: ["_bmad-output/planning-artifacts/dac-pipeline-product-brief.md"]
workflowType: architecture
project_name: ExpertFlow DaC Pipeline
user_name: Navira
date: '2026-05-12'
---

# Architecture Decision Document: ExpertFlow DaC Pipeline

---

## Lightweight Requirements

> This section replaces a formal PRD. The [product brief](https://github.com/expertflow/dac-finalization-plan/blob/master/dac-pipeline-product-brief.md) is the source of truth for goals and scope.
> If the brief changes, review this section first for impact.

### Functional Requirements

| # | Requirement |
|---|-------------|
| FR-01 | When a feature PR is opened, the pipeline generates a documentation draft automatically |
| FR-02 | The draft is generated from PR title, description, and code diff using Claude AI |
| FR-03 | The draft is committed as a markdown file to the PR branch — same repo as the code |
| FR-04 | A GitHub status check signals yellow (neutral) if the doc file is unmodified from the AI draft |
| FR-05 | The status check signals green (success) if the developer has edited the doc file |
| FR-06 | The status check never blocks a merge regardless of state |
| FR-07 | On merge to main, documentation is published to Docusaurus in draft state within 24 hours |
| FR-08 | Documentation coverage is tracked per feature and per team |
| FR-09 | A weekly coverage report is automatically sent to team leads every Monday morning |
| FR-10 | Every published doc records both the PR author (developer) and the PO who completed the async review |
| FR-11 | On merge, the pipeline auto-creates a GitHub Issue tagged `doc-review` in `expertflow-docs`, assigned to the responsible PO, with a direct link to the draft doc on Docusaurus |
| FR-12 | When the PO closes the `doc-review` Issue, a workflow updates the doc frontmatter `status` from `draft` to `published`, populates the `owner` field with the PO's GitHub handle, and triggers a Docusaurus rebuild that makes the doc customer-visible |

### Non-Functional Requirements

| # | Requirement |
|---|-------------|
| NFR-01 | Pipeline must be GitHub-native — no external CI/CD platforms |
| NFR-02 | Zero additional tooling required for developers beyond standard GitHub PR workflow |
| NFR-03 | Pipeline must handle PRs with no description gracefully (draft a minimal template, not fail) |
| NFR-04 | AI API calls must complete within 60 seconds to avoid GitHub Action timeouts |
| NFR-05 | Docusaurus draft publish must complete within 24 hours of merge |
| NFR-06 | Pipeline must be idempotent — re-running on the same PR must not duplicate docs |
| NFR-07 | Secrets (API keys) must never appear in logs or committed files |
| NFR-08 | Docs with `status: draft` must never be visible to external customers — Docusaurus must filter them from the public build |

### Constraints

- GitHub-native only — no Jenkins, GitLab, Bitbucket, or external CI
- Docusaurus is the publishing target — not Confluence
- Phase 1 scope: customer-facing feature docs only
- One pilot team leads by two weeks before full CX rollout
- Living document: brief changes trigger an architecture impact review of this document

---

## Project Context Analysis

### What We Are Building

The DaC Pipeline is not an application — it is a **GitHub Actions-based automation pipeline** that:

1. Intercepts pull requests as they are opened
2. Calls the Claude API to generate a documentation draft
3. Commits that draft into the PR branch
4. Reports a status check on the PR based on whether the draft was reviewed
5. On merge, triggers a Docusaurus rebuild and deployment (doc visible internally as `draft`)
6. On merge, creates a `doc-review` GitHub Issue assigned to the PO for async quality review
7. On PO approval (Issue close), flips doc to `published` state and rebuilds Docusaurus for customer visibility
8. On a weekly schedule, generates and distributes a team-level coverage report including PO review backlog

There is no user-facing UI. There is no database. There is no server to run or maintain. The entire system runs as GitHub Actions triggered by GitHub events.

### Complexity Assessment

| Dimension | Assessment |
|---|---|
| Primary domain | DevOps / CI automation |
| Complexity level | Medium — multiple GitHub Actions workflows, external API integration, cross-repo coordination |
| Real-time requirements | None |
| Multi-tenancy | No — each CX team repo runs its own instance of the pipeline |
| Regulatory compliance | None for Phase 1 |
| Integration complexity | Medium — GitHub API, Claude API, Docusaurus build, optional Slack/Teams webhook |
| Data volume | Low — markdown files, JSON coverage records |

### Technical Constraints Identified

- GitHub Actions free tier limits: 2,000 minutes/month for private repos (must monitor for pilot)
- Claude API: rate limits and token costs must be scoped (diff truncation strategy needed for large PRs)
- Docusaurus: must live in a dedicated docs repository, not inside every feature repo
- No shared database: coverage data stored as committed JSON files or GitHub API queries
- Docusaurus draft filtering: docs with `status: draft` must be excluded from the public sidebar and build output

### Cross-Cutting Concerns

- **Secret management:** `ANTHROPIC_API_KEY` must be available across all team repos — use GitHub Organization Secrets
- **Idempotency:** pipeline must not re-generate docs if already generated for a given PR SHA
- **Error handling:** AI API failures must not fail the PR status check in a blocking way — degrade gracefully
- **Repo configuration:** each team repo must opt in by adding the workflow file — covered during pilot onboarding
- **PO assignment:** a per-team PO handle mapping must be maintained (see `po-assignments.yml` config file below)

---

## Technology Stack

> No starter template applies — this is a pipeline, not an application. Technologies are chosen per component.

### Core Components and Technology Choices

| Component | Technology | Rationale |
|---|---|---|
| Pipeline runtime | GitHub Actions | GitHub-native, zero additional infrastructure |
| AI draft generation | Anthropic Claude API (`claude-sonnet-4-6`) | Best quality/cost balance for doc generation; Sonnet sufficient for structured markdown output |
| Doc format | Markdown (`.md`) | Docusaurus-compatible, editable in GitHub UI without tooling, renderable as PR preview |
| PR status check | GitHub Actions check runs (via `actions/github-script`) | Native GitHub integration, shows yellow/green on PR without external services |
| Publishing | Docusaurus (static site) | Established, ExpertFlow-chosen platform |
| Draft doc visibility | Docusaurus `custom_edit_url: null` + frontmatter filter plugin | Docs with `status: draft` excluded from public sidebar and build; visible only via direct URL to internal teams |
| Docs hosting | GitHub Pages or existing hosting | Determined by ExpertFlow infra — pipeline only triggers the build |
| Coverage tracking | Scheduled GitHub Action + GitHub API | No database needed; PRs and merged commits are the source of truth |
| PO review tracking | GitHub Issues (label: `doc-review`) | Native, no new tooling; PO assigned on Issue creation, ownership confirmed on Issue close |
| Weekly report delivery | GitHub Issue (auto-created, labeled `doc-coverage`) | GitHub-native, team leads notified via existing watch settings; optionally extended to Slack webhook |
| Secrets | GitHub Organization Secrets | Shared across all CX team repos from one place |

### What We Are Not Using

- No external CI (Jenkins, CircleCI, etc.)
- No database (PostgreSQL, MongoDB, etc.)
- No backend server or API layer
- No frontend application
- No Confluence (product docs go to Docusaurus only)
- No Docker (GitHub Actions handles compute)

---

## Core Architectural Decisions

### Decision 1: Two-Repo Model (Feature Repo + Docs Repo)

**Decision:** Documentation markdown files are generated into the feature repo (`docs/features/`) during the PR lifecycle, then synced to a dedicated `expertflow-docs` repository where Docusaurus lives and builds.

**Rationale:** Embedding Docusaurus setup (node_modules, config files, build tooling) into every CX feature repo creates maintenance burden and noise. A dedicated docs repo keeps Docusaurus configuration centralized while allowing docs to originate in the feature context where they belong.

**How it works:**
- Feature repo: `docs/features/` folder contains markdown files, committed alongside code
- On merge to main: a GitHub Action in the feature repo opens a PR to `expertflow-docs` with the new doc file (frontmatter `status: draft`)
- `expertflow-docs`: receives the PR, auto-merges (or lead reviews), triggers Docusaurus build and deploy — doc is live but draft-filtered (internal only)
- After PO closes `doc-review` Issue: `dac-po-publish.yml` updates `status: published` and rebuilds — doc becomes customer-visible

**Trade-off accepted:** One additional automated PR per merged feature, plus one Issue per merged feature for PO review. Both are auto-created; only the Issue requires a human action (PO closing it).

---

### Decision 2: AI Draft Committed to PR Branch (Not Posted as Comment)

**Decision:** The AI-generated draft is committed as a file (`docs/features/{feature-slug}.md`) directly to the feature PR branch by the GitHub Action bot, not posted as a PR comment.

**Rationale:** A committed file is editable, version-controlled, and merges with the code in the same commit. A comment is ephemeral, not version-controlled, and requires a secondary action to persist. Committing the file means the doc ships with the feature by default — the developer's only task is to edit it if the AI got something wrong.

**Feature slug derivation:** kebab-case of PR title, truncated to 60 characters, special characters stripped. Example: `"Add bulk export to CSV"` → `add-bulk-export-to-csv.md`

---

### Decision 3: Status Check Logic (Green / Yellow / Not Present)

**Decision:**
- **Green (success):** The `docs/features/{slug}.md` file exists in the PR AND has been modified by the developer (git diff shows developer edits beyond the bot's initial commit)
- **Yellow (neutral):** The file exists but is identical to the bot's initial commit — not yet reviewed
- **Missing (neutral):** The file was not generated (e.g., PR has no description and diff was empty) — a warning comment is posted but no blocking check

**Rationale:** Green rewards engagement. Yellow is visible but non-blocking. Missing is treated as a pipeline gap, not a developer failure — the bot failed, not the developer.

---

### Decision 4: Coverage Tracking via GitHub API (No Database)

**Decision:** Coverage is calculated on-demand by querying the GitHub API for merged PRs in a time window and checking whether each merged PR included a `docs/features/*.md` file.

**Rationale:** No external database to provision or maintain. The merged PR history is the authoritative record. Coverage is always computable from first principles.

**Coverage formula:**
```
coverage_rate = (PRs merged with docs/features/*.md) / (total PRs merged to main) × 100
```
PRs with label `skip-doc` are excluded from denominator (escape hatch for non-feature PRs like dependency bumps).

---

### Decision 5: Weekly Report as GitHub Issue

**Decision:** The weekly coverage report is created as a GitHub Issue in the `expertflow-docs` repository, labeled `doc-coverage-report`, tagged with the week date, and assigned to team leads via GitHub's @mention. The report also includes a **PO review backlog section** — docs currently in `status: draft` with an open `doc-review` Issue older than 5 business days.

**Rationale:** GitHub Issues are native, require no external tooling, are persistent and searchable, and team leads are already notified via GitHub watch settings. Surfacing overdue PO reviews in the same report avoids a separate notification channel and keeps accountability visible in one place.

---

### Decision 6: Claude API Prompt Strategy

**Decision:** The Claude API call uses a structured prompt that receives:
1. PR title
2. PR description (body)
3. Code diff — filtered to non-test, non-config file changes, truncated to 8,000 tokens maximum

**Output format:** Claude is instructed to produce a markdown file with a fixed template structure:
```
## Overview
## What's New
## How It Works
## Configuration (if applicable)
```

**Graceful degradation:** If PR description is empty and diff is also empty/trivial, Claude generates a minimal stub with placeholders rather than failing. The status check reports yellow (not red) in this case.

---

### Decision 7: Two-Stage Doc Visibility Model (Draft → Published)

**Decision:** All docs synced to `expertflow-docs` on merge land with `status: draft` in their frontmatter. Docusaurus is configured to exclude `status: draft` documents from the public sidebar and build. Docs become customer-visible only when a PO closes the associated `doc-review` Issue, triggering `dac-po-publish.yml` to flip the status to `published` and rebuild.

**Rationale:** This preserves the pipeline's core non-blocking principle (developers never wait, features merge freely) while ensuring customers only see PO-reviewed content. The PO's quality gate operates entirely post-merge and async — it is never in the critical path of feature delivery.

**Docusaurus implementation:** A Docusaurus plugin or `docusaurus.config.js` custom filter excludes documents where frontmatter `status !== 'published'` from the sidebar generation and static build output. Draft docs remain accessible via direct URL to internal teams.

**SLA:** Docs must reach `published` state within 5 business days of merge (tracked in weekly coverage report).

---

### Decision 8: PO Assignment via Config File

**Decision:** PO GitHub handles are maintained in a `po-assignments.yml` config file in `expertflow-docs`. The `dac-po-review.yml` workflow reads this file to determine which PO to assign for each team's `doc-review` Issue.

**Rationale:** Hardcoding PO handles in workflow files requires a workflow edit to change assignees. A config file separates the assignment data from the pipeline logic and can be updated by team leads without touching workflow YAML.

**Config format:**
```yaml
teams:
  cx-team-alpha:
    repo: expertflow/cx-alpha
    po: po-github-handle-alpha
  cx-team-beta:
    repo: expertflow/cx-beta
    po: po-github-handle-beta
```

---

## Implementation Patterns

### File Naming Conventions

| Artifact | Convention | Example |
|---|---|---|
| Feature doc file | `{kebab-case-pr-title}.md` | `add-bulk-export-to-csv.md` |
| GitHub Actions workflow — draft generation | `dac-draft-generator.yml` | `.github/workflows/dac-draft-generator.yml` |
| GitHub Actions workflow — doc sync | `dac-doc-sync.yml` | `.github/workflows/dac-doc-sync.yml` |
| GitHub Actions workflow — PO review Issue creation | `dac-po-review.yml` | `expertflow-docs/.github/workflows/dac-po-review.yml` |
| GitHub Actions workflow — PO publish trigger | `dac-po-publish.yml` | `expertflow-docs/.github/workflows/dac-po-publish.yml` |
| GitHub Actions workflow — coverage report | `dac-coverage-report.yml` | `expertflow-docs/.github/workflows/dac-coverage-report.yml` |
| PO assignment config | `po-assignments.yml` | `expertflow-docs/po-assignments.yml` |
| Coverage report issue title | `Doc Coverage Report — Week of YYYY-MM-DD` | `Doc Coverage Report — Week of 2026-05-12` |
| PO review issue title | `Doc Review: {feature-slug} [{team-name}]` | `Doc Review: add-bulk-export-to-csv [cx-team-alpha]` |
| Skip-doc label | `skip-doc` | Applied to PRs that should not count toward coverage |
| PO review label | `doc-review` | Applied to all PO review Issues |

### GitHub Action Trigger Conventions

| Workflow | Trigger | Condition |
|---|---|---|
| Draft generator | `pull_request: [opened, synchronize]` | Only on PRs targeting `main` or `master` |
| Doc sync | `push` to `main`/`master` | Only if `docs/features/**` changed |
| PO review Issue creation | `push` to `main` in `expertflow-docs` | Only if `docs/features/**` changed and `status: draft` |
| PO publish trigger | `issues: [closed]` | Only if closed Issue has label `doc-review` |
| Coverage report | `schedule: cron: '0 9 * * 1'` | Every Monday 09:00 UTC |

### Bot Commit Convention

The bot commits the draft file using the `GITHUB_TOKEN` with:
- Author: `DaC Bot <dac-bot@expertflow.com>`
- Commit message: `docs(auto): generate draft for PR #{pr_number}`
- The bot commit SHA is recorded in the PR's description as a hidden HTML comment to support idempotency checks

### Doc File Structure (Mandatory Template)

Every generated doc must follow this structure exactly:

```markdown
---
pr: {pr_number}
author: {pr_author}
owner: ''
date: {merge_date}
status: draft
---

# {PR Title}

## Overview

{1-2 sentence description of what this feature does}

## What's New

{Bullet list of changes visible to end users}

## How It Works

{Brief explanation of the mechanism, written for a non-developer reader}

## Configuration

{Only include if the feature introduces configurable options}
```

The `owner` field is empty at generation time. `dac-po-publish.yml` populates it with the PO's GitHub handle when the `doc-review` Issue is closed. The `status` field changes from `draft` to `published` at the same time.

### Error Handling Patterns

| Error | Behaviour |
|---|---|
| Claude API timeout (>60s) | Retry once; if fails again, commit a stub template file and set status check to yellow with message "AI draft unavailable — please write manually" |
| Claude API auth failure | Fail the Action with a clear error; alert via GitHub Action notification (not a PR status check failure) |
| PR has no description and empty diff | Commit a stub file with all sections as `[TBD]` placeholders |
| Bot commit fails (permissions) | Post a PR comment explaining the failure; do not set a blocking check |
| Doc sync PR to expertflow-docs fails | Post a comment on the merged feature PR; add to coverage gap report |
| PO not found in po-assignments.yml for team | Create `doc-review` Issue unassigned; flag in weekly coverage report as configuration gap |
| dac-po-publish.yml triggered but doc not found | Log error in Action; post comment on the closed Issue with failure details |

### Idempotency Pattern

Before generating a draft, the workflow checks:
```
git log --oneline --author="DaC Bot" -- docs/features/
```
If a bot commit already exists for the current PR head SHA (stored in PR description as `<!-- dac-sha: {sha} -->`), skip generation. This prevents duplicate docs on `synchronize` events for already-drafted PRs.

---

## Project Structure

### Feature Repository Structure

```
{feature-repo}/
├── .github/
│   └── workflows/
│       ├── dac-draft-generator.yml   # Triggered on PR open/sync — generates AI draft
│       └── dac-doc-sync.yml          # Triggered on merge — PRs doc to expertflow-docs
├── docs/
│   └── features/
│       ├── add-bulk-export-to-csv.md
│       ├── realtime-agent-status.md
│       └── ...                       # One file per merged feature
├── src/
│   └── ...                           # Existing codebase unchanged
└── README.md
```

### Docs Repository Structure (`expertflow-docs`)

```
expertflow-docs/
├── .github/
│   └── workflows/
│       ├── deploy.yml                # Triggered on push to main — builds + deploys Docusaurus
│       ├── dac-po-review.yml         # Triggered on merge — creates doc-review Issue for PO
│       ├── dac-po-publish.yml        # Triggered on doc-review Issue close — flips status, rebuilds
│       └── dac-coverage-report.yml  # Scheduled Monday 09:00 UTC — generates weekly report
├── docs/
│   ├── intro.md
│   └── features/
│       ├── add-bulk-export-to-csv.md
│       └── ...                       # Synced from feature repos
├── po-assignments.yml                # Team → PO GitHub handle mapping
├── docusaurus.config.js              # Includes draft-filter plugin config
├── sidebars.js
├── package.json
└── static/
```

### GitHub Organization Secrets

| Secret | Scope | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | Organization-wide | Claude API authentication for all CX team repos |
| `DOCS_REPO_PAT` | Organization-wide | Personal Access Token to allow feature repos to open PRs against `expertflow-docs` |
| `SLACK_WEBHOOK_URL` (optional) | `expertflow-docs` only | Weekly report Slack notification |

### Data Flow

```
Developer opens PR
        │
        ▼
dac-draft-generator.yml triggers
        │
        ├── Checks idempotency (bot commit exists?)
        │           │
        │    YES ───┴─── Skip, exit
        │    NO
        │
        ├── Calls Claude API with PR title + description + filtered diff
        │
        ├── Commits docs/features/{slug}.md to PR branch (bot author, status: draft)
        │
        └── Posts GitHub status check
                    │
                    ├── File exists + developer edits → GREEN
                    └── File exists, unmodified → YELLOW (neutral)

Developer merges PR
        │
        ▼
dac-doc-sync.yml triggers
        │
        ├── Checks docs/features/** for changed files
        │
        ├── Opens a PR to expertflow-docs with the new doc file (status: draft)
        │
        └── Auto-merges → triggers deploy.yml

deploy.yml triggers (expertflow-docs)
        │
        └── Builds Docusaurus → deploys (draft doc live but NOT customer-visible)

dac-po-review.yml triggers (on merge to expertflow-docs)
        │
        ├── Reads po-assignments.yml to resolve PO handle for the team
        │
        └── Creates GitHub Issue (label: doc-review) assigned to PO
                    └── Body includes: link to draft doc, feature name, team, merge date

PO reviews draft doc on Docusaurus (internal URL)
        │
        ├── If edits needed → raises a doc PR directly → Docusaurus rebuilds
        │
        └── Closes doc-review Issue when satisfied

dac-po-publish.yml triggers (on doc-review Issue close)
        │
        ├── Reads Issue body to resolve docs/features/{slug}.md path
        │
        ├── Updates frontmatter: status: draft → status: published
        │                        owner: '' → owner: {po_github_handle}
        │
        └── Commits change → triggers deploy.yml rebuild
                    └── Doc now customer-visible in Docusaurus public build

Every Monday 09:00 UTC
        │
        ▼
dac-coverage-report.yml triggers
        │
        ├── Queries GitHub API: merged PRs to main (past 7 days, per team repo)
        │
        ├── Checks each merged PR for docs/features/*.md inclusion
        │
        ├── Calculates coverage % per team and per developer
        │
        ├── Queries open doc-review Issues older than 5 business days → PO review backlog
        │
        └── Creates GitHub Issue in expertflow-docs (labeled doc-coverage-report)
                    └── @mentions team leads + flags overdue PO reviews by name
```

---

## Architecture Validation

### Coherence Validation

| Check | Status | Notes |
|---|---|---|
| All tech choices work together | ✅ | GitHub Actions + Claude API + Docusaurus are independent, composable |
| No contradictory decisions | ✅ | Two-repo model and same-repo docs are reconciled: docs originate in feature repo, sync to docs repo |
| Patterns align with technology stack | ✅ | YAML workflows, markdown files, GitHub API — all native to the chosen stack |
| Warn-not-block preserved end to end | ✅ | Status check uses `neutral` conclusion — GitHub does not block merge on neutral checks |
| PO review is never in delivery critical path | ✅ | PO review triggers post-merge — feature ships regardless of PO review state |
| Draft docs never customer-visible before PO approval | ✅ | Docusaurus draft filter + two-stage publish model enforces this |

### Requirements Coverage

| Requirement | Covered By |
|---|---|
| FR-01: Draft generated on PR open | `dac-draft-generator.yml` — `pull_request: opened` trigger |
| FR-02: Draft from PR title, description, diff | Claude API prompt in draft generator |
| FR-03: Draft committed to PR branch, same repo | Bot commit step in draft generator |
| FR-04: Yellow check if unmodified | Status check logic comparing bot commit SHA to current file SHA |
| FR-05: Green check if developer edited | Same status check — detects non-bot commits to `docs/features/{slug}.md` |
| FR-06: Never blocks merge | GitHub check `conclusion: neutral` — does not block |
| FR-07: Draft published within 24 hours | `deploy.yml` triggered immediately on merge to `expertflow-docs`; doc live but draft-filtered |
| FR-08: Coverage tracked per feature and team | `dac-coverage-report.yml` — GitHub API query per repo |
| FR-09: Weekly report to team leads | Scheduled workflow + GitHub Issue with @mentions |
| FR-10: Explicit doc ownership (author + PO owner) | `author` field set at generation; `owner` field set by `dac-po-publish.yml` on PO approval |
| FR-11: PO review Issue auto-created on merge | `dac-po-review.yml` — triggers on merge to `expertflow-docs`, reads `po-assignments.yml` |
| FR-12: PO Issue close triggers publish + owner assignment | `dac-po-publish.yml` — triggers on `issues: closed` with label `doc-review` |
| NFR-01: GitHub-native | ✅ 100% GitHub Actions |
| NFR-02: Zero additional tooling for devs | ✅ No installs, no new accounts, no new tools |
| NFR-03: Graceful handling of empty PRs | Stub template generation path |
| NFR-04: API calls complete within 60s | Diff truncation to 8,000 tokens; retry once on timeout |
| NFR-05: Draft publish within 24 hours | Docusaurus build is typically < 5 minutes; deploy is immediate |
| NFR-06: Idempotent | Bot commit SHA check before generation |
| NFR-07: Secrets never in logs | GitHub Actions masks secrets automatically; never echo API keys |
| NFR-08: Draft docs never customer-visible | Docusaurus draft-filter plugin excludes `status: draft` from public build |

### Gap Analysis

| Gap | Priority | Resolution |
|---|---|---|
| Docusaurus hosting target not specified | Important | Infrastructure team to confirm: GitHub Pages vs self-hosted. Pipeline is hosting-agnostic; only the deploy step changes |
| `skip-doc` label workflow not defined | Important | Needs a brief label governance decision — who can apply it, what qualifies |
| Auto-merge policy for expertflow-docs PRs | Important | Decide: auto-merge all doc sync PRs, or lead review required. Recommend auto-merge for Phase 1 with lead override option |
| PR description quality guidance for developers | Nice-to-have | A short "Good PR description = better AI draft" note in onboarding — not a pipeline concern |
| Pilot repo identification | Important | Confirm which CX team repo the pilot runs on before implementation begins |
| PO GitHub handles per team | Important | `po-assignments.yml` must be populated before `dac-po-review.yml` goes live; confirm handles with team leads during pilot onboarding |
| Docusaurus draft-filter plugin selection | Important | Two options: (a) community plugin `docusaurus-plugin-docs-draft`; (b) custom `docusaurus.config.js` filter. Confirm during `expertflow-docs` setup |

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION — pending resolution of Important gaps above (hosting target, auto-merge policy, PO handle config, draft-filter plugin selection)

**Confidence Level:** High

**Key Strengths:**
- Entirely GitHub-native — no new infrastructure to provision or maintain
- Zero developer learning curve — the pipeline adds files; developers do nothing new
- PO quality gate is fully async and post-merge — never blocks feature delivery
- Modular — each workflow file is independent; failures in one do not cascade to others
- Living and updatable — each section is self-contained; brief changes map cleanly to specific sections here

**Deferred to Phase 2+:**
- API documentation generation (different prompt strategy needed — code annotation parsing)
- Multi-repo aggregation optimizations (if CX team count grows beyond 10)
- Doc quality scoring (automated readability metrics on generated drafts)

### Implementation Handoff

**First implementation step:** Set up `expertflow-docs` repository with Docusaurus config, the `deploy.yml` workflow, and the draft-filter plugin. Populate `po-assignments.yml` with PO handles confirmed by team leads. This is the foundation everything else syncs to.

**Second step:** Add `dac-draft-generator.yml` to the pilot team's feature repository. Run one real PR through it end-to-end before onboarding other teams.

**Third step:** Add `dac-doc-sync.yml` to the pilot repo and validate the cross-repo PR flow to `expertflow-docs`. Confirm doc lands with `status: draft` and is not customer-visible.

**Fourth step:** Add `dac-po-review.yml` and `dac-po-publish.yml` to `expertflow-docs`. Run one end-to-end PO review cycle manually — create a test Issue, close it, confirm doc flips to `published` and Docusaurus rebuilds correctly.

**Fifth step:** Add `dac-coverage-report.yml` to `expertflow-docs` and validate the Monday report on a manual trigger before relying on the schedule. Confirm PO review backlog section surfaces correctly.

**All AI agents implementing this must:**
- Follow file naming conventions exactly as specified — slugs, workflow file names, frontmatter fields
- Never modify the `neutral` conclusion on the status check to `failure` — warn-not-block is a product decision, not a technical default
- Use `GITHUB_TOKEN` for in-repo commits and `DOCS_REPO_PAT` for cross-repo PRs — never the other way around
- Record bot commits with the `DaC Bot` author identity consistently across all repos
- Never publish a doc to the customer-visible Docusaurus build without `status: published` in its frontmatter
- Always populate the `owner` field from the PO who closed the `doc-review` Issue — never from the PR author
