# Framework Scoring Model

## Overview
This document defines the diagnostic questionnaire, scoring weights, and rationale outputs that power the project management framework selector. The model compares Scrum, Kanban, Scrumban, and optional frameworks (Extreme Programming and Lean Startup) across six delivery dimensions. Each response contributes weighted evidence toward or against a framework fit, culminating in a ranked recommendation and auditable rationale trail.

## Diagnostic Questions & Weights
| ID | Dimension | Diagnostic Question | Response Options | Scrum | Kanban | Scrumban | XP | Lean Startup |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `team_structure` | Team composition & stability | How stable and cross-functional is the delivery team? | Stable, cross-functional team with clear roles | +5 | +2 | +3 | +4 | +1 |
|  |  |  | Team composition changes frequently or relies on specialists | -3 | +4 | +2 | -2 | +1 |
|  |  |  | Highly distributed contributors with limited shared rituals | -2 | +3 | +1 | -3 | +2 |
| `cadence_commitment` | Cadence & commitment | What level of time-boxed commitment do stakeholders expect? | High: fixed-length increments with committed scope | +5 | -2 | +3 | +2 | -1 |
|  |  |  | Moderate: cadence desired but scope flexible | +2 | +2 | +4 | +1 | +2 |
|  |  |  | Low: continuous flow with minimal commitments | -3 | +5 | +2 | 0 | +3 |
| `work_variability` | Work item variability | How variable are work item sizes and arrival patterns? | Predictable sizes with planned intake | +4 | +1 | +3 | +3 | +1 |
|  |  |  | Mixed variability requiring re-planning | +1 | +3 | +4 | +1 | +2 |
|  |  |  | Highly variable with urgent items | -2 | +5 | +2 | 0 | +3 |
| `discovery_ratio` | Discovery vs delivery | What proportion of work is exploratory discovery? | >60% experiments and hypothesis testing | -1 | +1 | +2 | +2 | +5 |
|  |  |  | Balanced mix of discovery and delivery | +3 | +2 | +4 | +3 | +3 |
|  |  |  | >70% execution with clear requirements | +4 | +2 | +3 | +1 | -2 |
| `governance` | Governance & compliance | What level of governance is required? | Strict regulatory controls with audit trails | +2 | +1 | +3 | +1 | -3 |
|  |  |  | Moderate governance with pragmatic documentation | +4 | +2 | +4 | +2 | 0 |
|  |  |  | Lightweight governance, speed prioritized | +1 | +3 | +2 | +3 | +4 |
| `improvement_culture` | Continuous improvement | How mature is continuous improvement practice? | Mature, data-driven improvement rituals | +4 | +3 | +4 | +3 | +2 |
|  |  |  | Emerging culture with some metrics | +3 | +2 | +3 | +2 | +3 |
|  |  |  | Nascent rituals, limited metrics | +1 | +2 | +1 | +2 | +3 |

**Weight scale.** All weights follow a -5 to +5 scale stored in `config/scoring_rules.json`. Positive values increase fit, negative values penalize fit; higher magnitudes represent stronger evidence. Optional frameworks (XP, Lean Startup) stay in the scorecard but are only recommended when their net score exceeds a configurable trigger (default >10).

## Scoring Workflow
1. **Collect responses.** Facilitator captures one option per question.
2. **Apply weights.** Sum the weights for each framework using the JSON definitions. Missing responses default to zero weight.
3. **Normalize (optional).** When fewer than six questions are answered, normalize by `(score / number_of_answers) * total_questions` to keep ranges comparable.
4. **Rank frameworks.** Order by total score, highest first.
5. **Apply tie-breakers.**
   - **Dimension alignment.** Prefer the framework with more 4+ weights in stakeholder-priority dimensions.
   - **Optionality trigger.** If tie persists and an optional framework has any individual weight >3, flag for qualitative review instead of automatic selection.
   - **Human override.** Facilitator reviews rationale narratives with stakeholders to reach the final recommendation.
6. **Generate rationale.** Use templates below to explain positive and negative factors per framework, citing the relevant dimensions.

## Data Structures
The scoring service consumes `config/scoring_rules.json`:
- `version`, `last_reviewed`, and `default_weight_scale` support audit traceability.
- `frameworks` lists the supported options in display order.
- `questions` is an array of objects with `id`, `dimension`, `question`, and `options`.
- Each option defines a stable `id`, the display `label`, and a `weights` map assigning integers to every framework.
- `tie_breakers` enumerates deterministic resolution steps with `priority` ordering.

Future extensions can add `rationale_overrides` (per-option markdown snippets) or `constraints` (e.g., minimum team size) without breaking compatibility, as consumers read unknown fields permissively.

## Rationale Templates
Rationales are built by combining positive and negative fragments per dimension:
- **Positive fit:** "`<Framework>` aligns with your `<dimension>` needs because `<response paraphrase>` (weighted +3 or higher)."
- **Neutral fit:** "`<Framework>` is acceptable for `<dimension>` but consider `<mitigation>` (weights between -2 and +2)."
- **Negative fit:** "`<Framework>` may struggle with `<dimension>` given `<response paraphrase>`; consider `<alternative practice>` (weights ≤ -3)."

Each fragment references the originating question ID and option label for traceability. The facilitator stitches together the top three positive and top two negative drivers to form a concise narrative, appending any optional framework triggers for context.

## Expert Review & Audit Alignment
- **Domain workshops (2024-04-12).** Delivery coaches validated the question set against real transformation assessments; feedback added the `discovery_ratio` dimension and clarified governance wording.
- **Audit dry run (2024-04-26).** Portfolio governance partners reviewed rationale samples, requesting explicit tie-breaker documentation and normalization guidance.
- **Final sign-off (2024-05-01).** Agile CoE approved version `1.0.0`, confirming that generated rationales identify both positive and negative drivers and cite data sources, meeting auditability expectations.

Future revisions must update `last_reviewed` and attach meeting notes to preserve the audit trail.
