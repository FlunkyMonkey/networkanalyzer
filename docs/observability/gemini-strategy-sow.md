# GeminiCLI Strategic Review — Statement of Work

## Role Definition

**GeminiCLI acts as an external strategic advisor and milestone reviewer.** GeminiCLI does not own implementation, QA execution, or deployment decisions.

## Distinction from Other Roles

| Role | Owner | Scope |
|---|---|---|
| **Architecture authority** | ChatGPT | Architecture decisions, design review, approval of changes |
| **Implementation engine** | Claude Code | Code, manifests, docs, repo changes |
| **QA/regression contractor** | Codex | Test execution, regression checks, finding reports |
| **Strategic advisor** | GeminiCLI | Milestone review, risk assessment, strategic recommendations |
| **Approval and bridge** | Mike (operator) | Final go/no-go, communication, deployment execution |

## Scope

### In Scope

- Review completed phases for strategic alignment with project goals
- Assess whether the platform direction still serves the user's actual needs
- Identify risks, blind spots, or scope drift that the implementation team may have missed
- Evaluate the rollout plan for operational realism
- Review the testing model for completeness and coverage gaps
- Provide independent perspective on tradeoffs and priorities
- Recommend course corrections at milestone boundaries

### Out of Scope

- Implementation decisions (architecture authority is ChatGPT)
- Code changes or manifest modifications (implementation is Claude Code)
- Test execution or regression checks (QA is Codex)
- Deployment approval or go/no-go decisions (approval owner is Mike)
- Secret management or credential handling
- Day-to-day task prioritization

## Review Points

GeminiCLI reviews should occur at natural milestone boundaries:

| Milestone | What to Review |
|---|---|
| Post-Phase 7 (pre-deployment) | Is the platform ready for rollout? Are the docs sufficient? |
| Post-Wave 1 (telemetry live) | Is the telemetry plane delivering value? Any early course corrections? |
| Post-Wave 3 (flows live) | Is flow analytics meeting the top-talker/destination requirements? |
| Post-Wave 5 (go-live) | Does the full platform meet the original project goals? |
| Post-Phase 9 (optimization) | Is retention/rollup working? What's the long-term operational burden? |

## Deliverable Format

Each review should produce:

1. **Assessment:** Does the current state align with the project's stated goals?
2. **Risks:** What could go wrong that hasn't been addressed?
3. **Gaps:** What's missing or underspecified?
4. **Recommendations:** Specific, actionable suggestions (not vague "consider X")
5. **Priority call:** If time/effort is limited, what matters most right now?

## Communication Protocol

- GeminiCLI provides recommendations; it does not direct implementation
- Strategic recommendations are routed through Mike to the implementation team
- GeminiCLI does not communicate directly with Codex (QA) — findings flow through Mike
- If a GeminiCLI recommendation conflicts with an existing architecture decision, Mike resolves the conflict with ChatGPT (architecture authority)
- GeminiCLI reviews are asynchronous — they do not block active work unless Mike decides to pause
