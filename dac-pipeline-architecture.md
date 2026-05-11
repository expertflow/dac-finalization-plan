---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7]
inputDocuments: ["_bmad-output/planning-artifacts/dac-pipeline-product-brief.md"]
workflowType: architecture
project_name: ExpertFlow DaC Pipeline
user_name: Navira
date: '2026-05-11'
---

# Architecture Decision Document: ExpertFlow DaC Pipeline

---

## Lightweight Requirements

> This section replaces a formal PRD. The product brief is the source of truth for goals and scope.
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
| FR-07 | On merge to main, documentation is published to Docusaurus within 24 hours |
| FR-08 | Documentation coverage is tracked per feature and per team |
| FR-09 | A weekly coverage report is automatically sent to team leads every Monday morning |
| FR-10 | Every published doc has an explicit owner recorded (the PR author) |

### Non-Functional Requirements

| # | Requirement |
|---|-------------|
| NFR-01 | Pipeline must be GitHub-native — no external CI/CD platforms |
| NFR-02 | Zero additional tooling required for developers beyond standard GitHub PR workflow |
| NFR-03 | Pipeline must handle PRs with no description gracefully (draft a minimal template, not fail) |
| NFR-04 | AI API calls must complete within 60 seconds to avoid GitHub Action timeouts |
| NFR-05 | Docusaurus publish must complete within 24 hours of merge |
| NFR-06 | Pipeline must be idempotent — re-running on the same PR must not duplicate docs |
| NFR-07 | Secrets (API keys) must never appear in logs or committed files |

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
5. On merge, triggers a Docusaurus rebuild and deployment
6. On a weekly schedule, generates and distributes a team-level coverage report

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

### Cross-Cutting Concerns

- **Secret management:** `ANTHROPIC_API_KEY` must be available across all team repos — use GitHub Organization Secrets
- **Idempotency:** pipeline must not re-generate docs if already generated for a given PR SHA
- **Error handling:** AI API failures must not fail the PR status check in a blocking way — degrade gracefully
- **Repo configuration:** each team repo must opt in by adding the workflow file — covered during pilot onboarding

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
| Docs hosting | GitHub Pages or existing hosting | Determined by ExpertFlow infra — pipeline only triggers the build |
| Coverage tracking | Scheduled GitHub Action + GitHub API | No database needed; PRs and merged commits are the source of truth |
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
- On merge to main: a GitHub Action in the feature repo opens a PR to `expertflow-docs` with the new doc file
- `expertflow-docs`: receives the PR, auto-merges (or lead reviews), triggers Docusaurus build and deploy

**Trade-off accepted:** One additional automated PR per merged feature. This PR is auto-created and can be auto-merged via branch protection rules — no human action required for the standard path.

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

**Decision:** The weekly coverage report is created as a GitHub Issue in the `expertflow-docs` repository, labeled `doc-coverage-report`, tagged with the week date, and assigned to team leads via GitHub's @mention.

**Rationale:** GitHub Issues are native, require no external tooling, are persistent and searchable, and team leads are already notified via GitHub watch settings. An optional Slack webhook can be added later without changing the core architecture.

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

## Implementation Patterns

### File Naming Conventions

| Artifact | Convention | Example |
|---|---|---|
| Feature doc file | `{kebab-case-pr-title}.md` | `add-bulk-export-to-csv.md` |
| GitHub Actions workflow — draft generation | `dac-draft-generator.yml` | `.github/workflows/dac-draft-generator.yml` |
| GitHub Actions workflow — doc sync | `dac-doc-sync.yml` | `.github/workflows/dac-doc-sync.yml` |
| GitHub Actions workflow — coverage report | `dac-coverage-report.yml` | `expertflow-docs/.github/workflows/dac-coverage-report.yml` |
| Coverage report issue title | `Doc Coverage Report — Week of YYYY-MM-DD` | `Doc Coverage Report — Week of 2026-05-11` |
| Skip-doc label | `skip-doc` | Applied to PRs that should not count toward coverage |

### GitHub Action Trigger Conventions

| Workflow | Trigger | Condition |
|---|---|---|
| Draft generator | `pull_request: [opened, synchronize]` | Only on PRs targeting `main` or `master` |
| Doc sync | `push` to `main`/`master` | Only if `docs/features/**` changed |
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

The `status: draft` frontmatter field changes to `status: published` when the doc is merged to `expertflow-docs`.

### Error Handling Patterns

| Error | Behaviour |
|---|---|
| Claude API timeout (>60s) | Retry once; if fails again, commit a stub template file and set status check to yellow with message "AI draft unavailable — please write manually" |
| Claude API auth failure | Fail the Action with a clear error; alert via GitHub Action notification (not a PR status check failure) |
| PR has no description and empty diff | Commit a stub file with all sections as `[TBD]` placeholders |
| Bot commit fails (permissions) | Post a PR comment explaining the failure; do not set a blocking check |
| Doc sync PR to expertflow-docs fails | Post a comment on the merged feature PR; add to coverage gap report |

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
│       └── dac-coverage-report.yml  # Scheduled Monday 09:00 UTC — generates weekly report
├── docs/
│   ├── intro.md
│   └── features/
│       ├── add-bulk-export-to-csv.md
│       └── ...                       # Synced from feature repos
├── docusaurus.config.js
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
        ├── Commits docs/features/{slug}.md to PR branch (bot author)
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
        ├── Opens a PR to expertflow-docs with the new doc file
        │
        └── Auto-merges (or lead approves) → triggers deploy.yml

deploy.yml triggers (expertflow-docs)
        │
        └── Builds Docusaurus → deploys to hosting target (< 24 hours from merge)

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
        └── Creates GitHub Issue in expertflow-docs (labeled doc-coverage-report)
                    │
                    └── @mentions team leads in the issue body
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

### Requirements Coverage

| Requirement | Covered By |
|---|---|
| FR-01: Draft generated on PR open | `dac-draft-generator.yml` — `pull_request: opened` trigger |
| FR-02: Draft from PR title, description, diff | Claude API prompt in draft generator |
| FR-03: Draft committed to PR branch, same repo | Bot commit step in draft generator |
| FR-04: Yellow check if unmodified | Status check logic comparing bot commit SHA to current file SHA |
| FR-05: Green check if developer edited | Same status check — detects non-bot commits to `docs/features/{slug}.md` |
| FR-06: Never blocks merge | GitHub check `conclusion: neutral` — does not block |
| FR-07: Published within 24 hours | `deploy.yml` triggered immediately on merge to `expertflow-docs` |
| FR-08: Coverage tracked per feature and team | `dac-coverage-report.yml` — GitHub API query per repo |
| FR-09: Weekly report to team leads | Scheduled workflow + GitHub Issue with @mentions |
| FR-10: Explicit doc owner recorded | `author` field in doc frontmatter (PR author) |
| NFR-01: GitHub-native | ✅ 100% GitHub Actions |
| NFR-02: Zero additional tooling for devs | ✅ No installs, no new accounts, no new tools |
| NFR-03: Graceful handling of empty PRs | Stub template generation path |
| NFR-04: API calls complete within 60s | Diff truncation to 8,000 tokens; retry once on timeout |
| NFR-05: Publish within 24 hours | Docusaurus build is typically < 5 minutes; deploy is immediate |
| NFR-06: Idempotent | Bot commit SHA check before generation |
| NFR-07: Secrets never in logs | GitHub Actions masks secrets automatically; never echo API keys |

### Gap Analysis

| Gap | Priority | Resolution |
|---|---|---|
| Docusaurus hosting target not specified | Important | Infrastructure team to confirm: GitHub Pages vs self-hosted. Pipeline is hosting-agnostic; only the deploy step changes |
| `skip-doc` label workflow not defined | Important | Needs a brief label governance decision — who can apply it, what qualifies |
| Auto-merge policy for expertflow-docs PRs | Important | Decide: auto-merge all doc sync PRs, or lead review required. Recommend auto-merge for Phase 1 with lead override option |
| PR description quality guidance for developers | Nice-to-have | A short "Good PR description = better AI draft" note in onboarding — not a pipeline concern |
| Pilot repo identification | Important | Confirm which CX team repo the pilot runs on before implementation begins |

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION — pending resolution of Important gaps above (hosting target + auto-merge policy)

**Confidence Level:** High

**Key Strengths:**
- Entirely GitHub-native — no new infrastructure to provision or maintain
- Zero developer learning curve — the pipeline adds files; developers do nothing new
- Modular — each workflow file is independent; failures in one do not cascade to others
- Living and updatable — each section is self-contained; brief changes map cleanly to specific sections here

**Deferred to Phase 2+:**
- API documentation generation (different prompt strategy needed — code annotation parsing)
- Multi-repo aggregation optimizations (if CX team count grows beyond 10)
- Doc quality scoring (automated readability metrics on generated drafts)

### Implementation Handoff

**First implementation step:** Set up `expertflow-docs` repository with Docusaurus config and the `deploy.yml` workflow. This is the foundation everything else syncs to.

**Second step:** Add `dac-draft-generator.yml` to the pilot team's feature repository. Run one real PR through it end-to-end before onboarding other teams.

**Third step:** Add `dac-doc-sync.yml` to the pilot repo and validate the cross-repo PR flow to `expertflow-docs`.

**Fourth step:** Add `dac-coverage-report.yml` to `expertflow-docs` and validate the Monday report on a manual trigger before relying on the schedule.

**All AI agents implementing this must:**
- Follow file naming conventions exactly as specified — slugs, workflow file names, frontmatter fields
- Never modify the `neutral` conclusion on the status check to `failure` — warn-not-block is a product decision, not a technical default
- Use `GITHUB_TOKEN` for in-repo commits and `DOCS_REPO_PAT` for cross-repo PRs — never the other way around
- Record bot commits with the `DaC Bot` author identity consistently across all repos
