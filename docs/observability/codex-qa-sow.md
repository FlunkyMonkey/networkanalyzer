# Codex QA — Statement of Work

## Role Definition

**OpenAI Codex acts as an independent QA/regression contractor.** Codex does not own architecture decisions, rollout approvals, or deployment execution.

## Scope

### In Scope

- Execute regression tests defined in [regression-test-plan.md](regression-test-plan.md)
- Execute operator test procedures from [operator-test-procedures.md](operator-test-procedures.md) where they can be verified from repo state or cluster access
- Validate that manifests render correctly: `kustomize build --enable-helm platform/overlays/lab`
- Validate markdown documentation lint: `markdownlint docs/**/*.md`
- Verify no secrets or credentials are committed to the repository
- Verify dashboard JSON is syntactically valid
- Report findings in the defined format (see below)
- Flag regressions between waves
- Verify rollout checklist items from [rollout-checklists.md](rollout-checklists.md) that can be validated without live cluster access

### Out of Scope

- Architecture decisions or changes
- Product selection or replacement
- Rollout approval (go/no-go decisions)
- Secret creation or credential management
- Live cluster deployment or cutover execution
- CNI migration
- Modifying manifests to fix issues (report only; Claude Code or the operator fixes)
- Approving Phase gates

## Reporting Format

Each finding must include:

| Field | Description |
|---|---|
| **ID** | Sequential identifier (e.g., QA-001) |
| **Severity** | Critical / High / Medium / Low / Info |
| **Category** | Build, Manifest, Documentation, Dashboard, Regression, Security |
| **Summary** | One-line description |
| **Evidence** | Command output, file path, or screenshot showing the issue |
| **Expected** | What should have happened |
| **Actual** | What actually happened |
| **Recommendation** | Suggested fix (optional) |

## Severity Definitions

| Severity | Definition | Action Required |
|---|---|---|
| **Critical** | Kustomize build fails. Secrets found in repo. Deployment would break. | Must fix before any rollout wave. |
| **High** | Dashboard does not load. Data source misconfigured. Key acceptance test would fail. | Must fix before the relevant rollout wave. |
| **Medium** | Documentation inconsistency. Minor dashboard display issue. Non-blocking lint warning. | Fix before go-live (Wave 5). |
| **Low** | Cosmetic issue. Minor wording improvement. Non-functional concern. | Fix opportunistically. |
| **Info** | Observation. Suggestion. No action required. | Log for awareness. |

## Evidence Requirements

- **Build issues:** Include the full error output from `kustomize build` or `helm template`
- **Dashboard issues:** Include the dashboard UID, panel title, and the specific error or rendering issue
- **Documentation issues:** Include the file path, line number, and what is incorrect
- **Security issues:** Include the file path and the nature of the exposure
- **Regression issues:** Reference the specific regression check (e.g., R2.3) and include before/after evidence

## Test Execution Order

1. **Build validation** — run `kustomize build` and `markdownlint` first
2. **Manifest review** — check for secrets, misconfigurations, schema issues
3. **Dashboard JSON validation** — verify all dashboard ConfigMaps contain valid JSON
4. **Documentation cross-reference** — verify docs reference correct file paths, dashboard UIDs, and commands
5. **Regression matrix** — run applicable regression checks from the regression test plan

## Pass/Fail Expectations

| Category | Pass Threshold |
|---|---|
| Build validation | Zero failures |
| Secret scan | Zero secrets in repo |
| Dashboard JSON | All valid |
| Documentation consistency | No Critical or High findings |
| Regression checks | All checks from the relevant wave pass |

## Deliverable

A single QA report document containing:

1. Summary: total findings by severity
2. Finding list in the format above
3. Pass/fail verdict per test category
4. Recommendation: proceed to rollout or remediate first

## Communication Protocol

- Codex reports findings; it does not fix them
- Critical findings are reported immediately
- The operator and Claude Code triage and prioritize fixes
- Codex re-tests after fixes are applied
- Go/no-go decisions are made by the operator (Mike), not Codex
