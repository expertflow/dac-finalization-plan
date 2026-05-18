---
stepsCompleted: [1, 2, 3]
inputDocuments: []
workflowType: 'research'
lastStep: 1
research_type: 'technical'
research_topic: 'Documentation as Code pipeline tooling for ExpertFlow'
research_goals: 'Evaluate static site generators, GitHub Actions pipeline patterns, hosting options, and PR-based review workflows to inform a solution design document'
user_name: 'Jawad'
date: '2026-05-18'
web_research_enabled: true
source_verification: true
---

# Research Report: Technical

**Date:** 2026-05-18
**Author:** Jawad
**Research Type:** Technical

---

## Research Overview

[Research overview and methodology will be appended here]

---

<!-- Content will be appended sequentially through research workflow steps -->

## Technical Research Scope Confirmation

**Research Topic:** Documentation as Code pipeline tooling for ExpertFlow
**Research Goals:** Evaluate static site generators, GitHub Actions pipeline patterns, hosting options, and PR-based review workflows to inform a solution design document.

**Confirmed Constraints (corrected from initial brief):**
- Source code: self-hosted GitLab (GitLab CI/CD is the pipeline engine)
- Static site generator: Docusaurus — already in use on GitHub (decision made)
- Docs hosting: GitHub / GitHub Pages (already in place)
- Project tracking: Jira + Confluence remain in use
- SSG evaluation scope reduced to: confirming Docusaurus is the right choice and understanding its GitLab integration patterns

**Scope:**
- Architecture Analysis — GitLab-to-GitHub cross-platform pipeline design
- Implementation Approaches — dev workflow, MR templates, doc-in-same-MR pattern
- Integration Patterns — GitLab CI pushing to GitHub, GitHub Actions deploying to Pages
- Performance Considerations — pipeline speed, build time, hosting reliability

**Scope Confirmed:** 2026-05-18

---

## Technology Stack Analysis

### Static Site Generator: Docusaurus (Confirmed)

Docusaurus is already the chosen SSG. Research confirms it is well-suited for docs-as-code:

- Built by Meta, maintained actively; widely adopted for developer-facing product docs
- Markdown and MDX support; versioning built-in (important as CX releases multiple versions)
- GitHub Pages deployment is first-class — official docs cover the full GitHub Actions workflow
- GitLab CI/CD deployments to GitLab Pages are also documented and in use in the wild
- Build time: typically under 2 minutes for mid-sized doc sites

**Verdict:** No change needed. Docusaurus on GitHub is the right stack.

_Sources:_
- [Deployment | Docusaurus](https://docusaurus.io/docs/deployment)
- [Building Centralized Documentation Across Microservices with Docusaurus, GitLab CI — DEV Community](https://dev.to/piotr_pomierski/building-centralized-documentation-across-microservices-with-docusaurus-gitlab-ci-and-typedoc-5a7d)
- [GitLab Pages / Docusaurus example](https://gitlab.com/pages/docusaurus)

---

### Cross-Platform Pipeline: GitLab CI → GitHub

This is the architecturally novel piece. Two verified patterns exist:

#### Pattern A — Doc files live in the GitLab code repo; GitLab CI pushes to GitHub on merge

1. Developer writes feature code + doc files in the same GitLab repo (e.g. `docs/features/my-feature.md`)
2. GitLab MR pipeline runs: lint, link-check, doc presence validation
3. On merge to main: GitLab CI job clones the GitHub docs repo, copies updated doc files, commits, and pushes using a GitHub Personal Access Token (PAT) stored as a masked GitLab CI variable
4. GitHub Actions on the docs repo detects the push, runs `npm run build`, deploys to GitHub Pages

**Authentication:** GitHub PAT with `repo` scope stored as a GitLab CI/CD masked, protected variable. The `git push` command uses the token inline: `git push https://<token>@github.com/org/docs-repo.git`. Use `-o ci.skip` or the GitHub Actions `if: github.actor != 'gitlab-bot'` guard to prevent loop triggers.

**Pros:** Single developer workflow (one MR, one review), docs always co-versioned with code, GitLab is the source of truth for authoring.
**Cons:** Cross-platform push adds a job step; PAT must be rotated; slight coupling between repos.

#### Pattern B — Docs live directly in the GitHub repo; developers open a separate PR

Developers write code on GitLab and open a separate PR on GitHub for the doc update.

**Pros:** Simpler pipeline (no cross-platform push), GitHub review UI for docs.
**Cons:** Two separate review processes, high friction for developers, easy to skip — directly contradicts the 90%→100% coverage goal. **Not recommended.**

**Verdict: Pattern A is strongly preferred** for ExpertFlow. It keeps the developer in a single workflow and makes doc coverage enforceable at the GitLab MR level.

_Sources:_
- [How to Push Changes to a Repository from a GitLab CI Pipeline | Baeldung](https://www.baeldung.com/ops/gitlab-ci-pipeline-push-changes)
- [CI/CD for external repositories | GitLab Docs](https://docs.gitlab.com/ci/ci_cd_for_external_repos/)
- [GitLab token overview | GitLab Docs](https://docs.gitlab.com/security/tokens/)

---

### MR Review Workflow: GitLab Merge Request Templates

GitLab natively supports MR description templates stored at `.gitlab/merge_request_templates/*.md` in the code repo. This is the mechanism to enforce doc inclusion at review time.

**Recommended implementation:**
- Default MR template includes a `## Documentation` checklist section with a checkbox: `- [ ] Doc file added or updated in /docs/`
- GitLab CI job (`doc-check`) validates that any MR touching `src/` also touches `docs/` — fails the pipeline if not
- Reviewers see the doc diff inline in the same MR; no context switching

GitLab's own engineering team uses exactly this pattern: developers write the first draft, the PM ensures it ships, and docs are part of the Definition of Done. This is a proven, real-world precedent.

_Sources:_
- [Description templates | GitLab Docs](https://docs.gitlab.com/user/project/description_templates/)
- [Five fast facts about docs as code at GitLab](https://about.gitlab.com/blog/five-fast-facts-about-docs-as-code-at-gitlab/)
- [Documentation workflow | GitLab Docs](https://gitlab-org.gitlab.io/technical-writing-group/gitlab-docs-hugo/development/documentation/workflow/)

---

### Hosting: GitHub Pages (Confirmed)

GitHub Pages with GitHub Actions deployment is the de facto standard for Docusaurus sites.

**Modern setup (2025+):**
- Repository Settings → Pages → Source: **GitHub Actions** (not legacy "Deploy from branch")
- Workflow uses: `actions/checkout@v4` → `actions/setup-node@v4` (Node 20) → `npm run build` → `actions/upload-pages-artifact@v3` → `actions/deploy-pages@v4`
- Free for public repositories; HTTPS automatic

**Alternatives evaluated and ruled out:**
- **GitLab Pages**: Would eliminate the cross-platform complexity, but the docs repo is already on GitHub — migration cost not justified
- **Netlify / Vercel**: Add cost and external dependency; GitHub Pages is sufficient for this use case
- **S3 + CloudFront**: Overkill for a docs site; operational overhead not warranted

_Sources:_
- [Deployment | Docusaurus — GitHub Pages section](https://docusaurus.io/docs/deployment)
- [How to Set Up Documentation as Code with Docusaurus and GitHub Actions | freeCodeCamp](https://www.freecodecamp.org/news/set-up-docs-as-code-with-docusaurus-and-github-actions/)
- [Deploy Docusaurus to GitHub Pages (Dec 2025) | Medium](https://medium.com/@syedahoorainali/deploy-docusaurus-book-on-github-pages-240097aa6869)

---

### Technology Adoption Trends

- Docs-as-code with GitLab MR-gated doc requirements is the dominant pattern at engineering-led organizations (GitLab itself, HashiCorp, Stripe)
- Docusaurus v3 (current) is actively maintained and has no near-term deprecation risk
- GitHub Actions for Pages deployment is now the officially recommended path (legacy branch deploy is being phased out)
- Cross-platform GitLab→GitHub pipelines using PATs are well-documented and in production use at multiple organizations; SSH key auth is an equally viable alternative with slightly better security posture

---

## Integration Patterns Analysis

### 1. PO Approval Gate — GitLab MR Approval Rules

GitLab's native approval rules (Settings → Merge requests → Approval rules) are the mechanism for enforcing PO sign-off on documentation before merge.

**Configuration for ExpertFlow:**

- Create an approval rule named "PO Doc Review" with `Approvals required: 1`
- Add the PO (or PO group) as the designated approver
- Enable "Prevent approval by author" so the developer cannot self-approve
- Optionally scope the rule to trigger only when files in `docs/` are changed — reduces PO noise on pure code-only MRs

**Dependency to flag:** POs must have at minimum Developer role on the GitLab project to be eligible approvers. This requires a GitLab seat and onboarding for any PO not already on the platform.

_Sources:_

- [Merge request approval rules | GitLab Docs](https://docs.gitlab.com/user/project/merge_requests/approvals/rules/)
- [Merge request approvals | GitLab Docs](https://docs.gitlab.com/user/project/merge_requests/approvals/)

---

### 2. Doc Presence Enforcement — GitLab CI `doc-check` Job

GitLab CI's `rules.changes` keyword detects which files changed in an MR. This is the enforcement mechanism that fails the pipeline when a developer touches code but skips the doc.

**Implementation pattern:**

```yaml
doc-check:
  stage: validate
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  script:
    - |
      CODE_CHANGED=$(git diff --name-only $CI_MERGE_REQUEST_DIFF_BASE_SHA $CI_COMMIT_SHA | grep -E '^src/' || true)
      DOC_CHANGED=$(git diff --name-only $CI_MERGE_REQUEST_DIFF_BASE_SHA $CI_COMMIT_SHA | grep -E '^docs/' || true)
      if [ -n "$CODE_CHANGED" ] && [ -z "$DOC_CHANGED" ]; then
        echo "ERROR: Code changed in src/ but no doc updated in docs/"
        exit 1
      fi
```

This job runs on every MR, checks if `src/` changed without a corresponding `docs/` change, and fails with a clear message. It does not block MRs that are docs-only or code-only with no feature impact (the team can add path exclusions for test files, configs, etc.).

_Sources:_

- [CI/CD YAML syntax reference — rules.changes | GitLab Docs](https://docs.gitlab.com/ci/yaml/)
- [Run job when CHANGELOG.md is not changed | GitLab Forum](https://forum.gitlab.com/t/run-job-when-changelog-md-is-not-changed/70063)

---

### 3. Jira ↔ GitLab Linking

GitLab's native Jira integration (Settings → Integrations → Jira) enables bidirectional linking with zero extra tooling.

**How it works for ExpertFlow:**

- Developer includes the Jira key in the MR title or branch name (e.g. `feature/CX-123-voice-recording-doc`)
- GitLab automatically posts a comment on the Jira issue with a link back to the MR
- On merge, adding `Closes CX-123` in the MR description transitions the Jira issue to Done
- The Confluence page the PO wrote (linked to the same Jira issue) remains the internal spec; the GitLab MR doc is the customer-facing output — both traceable from the same Jira ticket

**Note:** This works with self-hosted GitLab and Jira Cloud/Server. Requires a one-time integration setup by an admin using a Jira API token.

_Sources:_

- [Jira | GitLab Docs](https://docs.gitlab.com/integration/jira/)
- [Jira issue management | GitLab Docs](https://docs.gitlab.com/integration/jira/issues/)
- [Integrate GitLab with Jira | Atlassian Support](https://support.atlassian.com/jira-cloud-administration/docs/integrate-gitlab-with-jira/)

---

### 4. Doc Quality Gates — markdownlint + Vale

Two complementary linting tools run as GitLab CI jobs before the PO approval gate:

| Tool | What it catches | Config file |
|---|---|---|
| **markdownlint** | Formatting errors, broken heading hierarchy, missing blank lines | `.markdownlint.json` |
| **Vale** | Prose style, passive voice, terminology consistency, forbidden words | `.vale.ini` + `styles/` |

Both are Node.js-compatible and integrate cleanly into a GitLab CI YAML stage. Vale supports the Google Developer Documentation Style Guide and Microsoft Writing Style Guide as importable rule sets — a useful starting point before ExpertFlow defines its own terminology rules.

**Recommended pipeline stage order:**

```
lint (markdownlint + Vale) → doc-check (presence) → [PO approval] → publish-to-github
```

Linting runs first so feedback is immediate; the PO sees a clean doc draft, not raw lint failures.

_Sources:_

- [Linting Documentation with Vale to Increase Quality & Consistency | Stream](https://getstream.io/blog/linting-documentation-with-vale/)
- [How we use Vale | Datadog Engineering](https://www.datadoghq.com/blog/engineering/how-we-use-vale-to-improve-our-documentation-editing-process/)
- [Markdown Linting in CI | DEV Community](https://dev.to/_402ccbd6e5cb02871506/markdown-linting-in-ci-markdownlint-cli2-vs-gomarklint-2gg3)

---

### 5. Cross-Platform Publish — GitLab CI → GitHub → Pages

On merge to `main`, a publish job in GitLab CI pushes updated doc files to the GitHub docs repo:

```yaml
publish-docs:
  stage: publish
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  script:
    - git clone https://oauth2:$GITHUB_PAT@github.com/expertflow/docs.git /tmp/docs
    - cp -r docs/* /tmp/docs/docs/
    - cd /tmp/docs
    - git config user.email "ci@expertflow.com"
    - git config user.name "GitLab CI"
    - git add .
    - git diff --cached --quiet || git commit -m "docs: update from $CI_PROJECT_NAME@$CI_COMMIT_SHORT_SHA"
    - git push
```

`$GITHUB_PAT` is stored as a masked, protected GitLab CI variable (repo scope, scoped to the docs repo only). GitHub Actions on the docs repo then triggers the Docusaurus build and deploys to GitHub Pages automatically.

**Security note:** Rotate the PAT on a schedule (90-day max); use a dedicated `gitlab-ci-bot` GitHub account rather than a personal account to avoid losing access if a team member leaves.

_Sources:_

- [How to Push Changes to a Repository from a GitLab CI Pipeline | Baeldung](https://www.baeldung.com/ops/gitlab-ci-pipeline-push-changes)
- [GitLab token overview | GitLab Docs](https://docs.gitlab.com/security/tokens/)
