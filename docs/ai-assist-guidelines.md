# AI Assistance Operational Guidelines

This document defines how the AI-assisted planning module should interact with product managers, including approved prompt templates, escalation paths, guardrails for generated outputs, observability requirements, and latency management practices.

## 1. Approved Prompt Templates

Use the following prompt templates when requesting AI support. Each template contains a required **System Prompt**, contextual **User Input**, and optional **Follow-up Questions**.

### 1.1 Opportunity Assessment Prompt
- **System Prompt:**
  > You are an experienced product strategist. Provide structured opportunity assessments including customer problem statements, data-backed impact estimates, and key uncertainties.
- **User Input Skeleton:**
  ```
  Context: <market segment or product area>
  Signals: <metrics, research highlights>
  Decision Horizon: <timeframe>
  Constraints: <regulatory, technical, budget>
  ```
- **Follow-up Questions:** Clarify missing metrics, request risk ratings, or expand on dependency risks.

### 1.2 Roadmap Scenario Exploration Prompt
- **System Prompt:**
  > You are advising a roadmap review. Compare scenarios using cost/benefit, resource load, and strategic alignment. Return outputs as a table followed by key insights.
- **User Input Skeleton:**
  ```
  Scenarios:
    - Name: <scenario name>
      Inputs: <feature list, resources>
    - ...
  KPIs: <target metrics>
  Time Horizon: <quarter/year>
  ```
- **Follow-up Questions:** Ask for alternative sequencing, integration risks, or data confidence levels.

### 1.3 Post-Mortem Synthesis Prompt
- **System Prompt:**
  > You are a neutral facilitator summarizing a project post-mortem. Produce themes, quantified impact, and recommended guardrails.
- **User Input Skeleton:**
  ```
  Incident Summary: <what happened>
  Metrics Impacted: <KPI deltas>
  Root Cause Notes: <bullets>
  Stakeholder Quotes: <optional>
  ```
- **Follow-up Questions:** Request missing metrics, conflicting narratives, or preventative actions.

### 1.4 Custom Prompts
- Product managers may deviate from the templates only if:
  1. The request is exploratory and cannot fit the templates without losing critical context.
  2. The custom prompt is reviewed by a lead PM or AI safety steward before use.

## 2. Escalation Rules

1. **Safety & Compliance Escalation**
   - Trigger: AI output references personally identifiable information (PII), proposes regulatory violations, or contradicts published compliance policies.
   - Action: Immediately halt AI usage for the task, notify the AI safety steward, and file an incident ticket within 24 hours.

2. **Quality Escalation**
   - Trigger: AI recommendations conflict with quantitative evidence or appear logically inconsistent.
   - Action: Escalate to the product strategy lead for manual review. Document discrepancies in the decision log before proceeding.

3. **Latency Escalation**
   - Trigger: AI response exceeds 1.5 seconds twice within an hour or misses the 2-second SLA even once for a high-priority review.
   - Action: Switch to rule-based heuristics for the active session and alert the platform team to investigate throughput issues.

## 3. Guardrails for AI Outputs

- **Scope Guardrails:** Restrict AI to descriptive and comparative analysis. Final prioritization decisions must be rule-based or human-reviewed.
- **Data Guardrails:**
  - Never send raw customer PII or confidential financial forecasts to the AI model.
  - Provide only aggregated metrics or anonymized insights.
- **Tone & Format Guardrails:**
  - Require numbered recommendations with rationale and confidence levels.
  - Reject outputs containing sensitive language, speculation about individuals, or unverifiable claims.
- **Traceability Guardrails:**
  - Attach a unique `ai_interaction_id` to each prompt/response pair.
  - Store the prompt, response, and rule-based override rationale for 12 months.

## 4. Observability: Logging AI Suggestions and Final Recommendations

### 4.1 Data Structures

```sql
-- Table storing each AI interaction
CREATE TABLE ai_interactions (
  ai_interaction_id UUID PRIMARY KEY,
  request_timestamp TIMESTAMPTZ NOT NULL,
  requester_id TEXT NOT NULL,
  prompt_template TEXT NOT NULL,
  prompt_payload JSONB NOT NULL,
  response_payload JSONB NOT NULL,
  response_latency_ms INTEGER NOT NULL,
  model_version TEXT NOT NULL,
  safety_flag BOOLEAN DEFAULT FALSE,
  notes TEXT
);

-- Table linking AI suggestions to rule-based recommendations
CREATE TABLE recommendation_decisions (
  decision_id UUID PRIMARY KEY,
  ai_interaction_id UUID REFERENCES ai_interactions(ai_interaction_id),
  finalized_timestamp TIMESTAMPTZ NOT NULL,
  reviewer_id TEXT NOT NULL,
  rule_engine_version TEXT NOT NULL,
  final_recommendation JSONB NOT NULL,
  applied_guardrails JSONB NOT NULL,
  override_reason TEXT,
  decision_confidence NUMERIC(3,2) NOT NULL,
  audit_status TEXT DEFAULT 'pending'
);
```

#### Key Enumerations
- `prompt_template`: (`"opportunity_assessment"`, `"roadmap_scenario"`, `"post_mortem"`, `"custom"`).
- `audit_status`: (`"pending"`, `"approved"`, `"rejected"`).

### 4.2 API Endpoints

| Method | Endpoint | Description | Request Body | Response |
| --- | --- | --- | --- | --- |
| `POST` | `/api/v1/ai-interactions` | Log a new AI prompt/response pair. | `{ ai_interaction_id, requester_id, prompt_template, prompt_payload, response_payload, response_latency_ms, model_version, safety_flag?, notes? }` | `201 Created` with `{ ai_interaction_id }` |
| `GET` | `/api/v1/ai-interactions/{id}` | Retrieve a specific AI interaction. | `n/a` | `200 OK` with the full interaction record |
| `POST` | `/api/v1/recommendations` | Store the final rule-based recommendation referencing an AI suggestion. | `{ decision_id, ai_interaction_id, reviewer_id, rule_engine_version, final_recommendation, applied_guardrails, override_reason?, decision_confidence }` | `201 Created` with `{ decision_id }` |
| `GET` | `/api/v1/recommendations/{id}` | Retrieve a decision and its linked AI interaction metadata. | `n/a` | `200 OK` with joined record |
| `GET` | `/api/v1/recommendations?audit_status=pending` | List pending audits. | `n/a` | `200 OK` with an array of `decision_id` + metadata |

### 4.3 Event Streaming (Optional)
- Emit `ai_interaction.created` and `recommendation.finalized` events for downstream analytics and compliance monitoring.
- Include latency, safety flags, and override reasons as event attributes.

## 5. Latency Budgeting & Fallback Strategy

- **Target:** <2 seconds end-to-end per AI-assisted recommendation query.
- **Budget Allocation:**
  - 100 ms: Input validation & template lookup.
  - 400 ms: Prompt enrichment (fetching metrics, anonymization).
  - 1,100 ms: AI model inference (p95 target).
  - 200 ms: Post-processing, guardrail enforcement, logging.
  - 200 ms buffer: Network variability and retries.

- **Monitoring:**
  - Capture latency percentiles via tracing for each interaction.
  - Trigger alerts when p95 > 1.6 s over 15-minute windows or any single response exceeds 2 s.

- **Fallback Procedures:**
  1. **Graceful Degradation (1.6–2.0 s):** Return partial AI output (e.g., top insights) while concurrently running rule-based heuristics; inform the user that the full narrative is pending.
  2. **Hard Timeout (>2.0 s):** Cancel the AI call, surface rule-based recommendation only, and log `safety_flag = true` with reason `"latency_timeout"`.
  3. **Adaptive Sampling:** Reduce prompt context (drop non-critical metrics) when the system detects elevated latency over the last five requests.
  4. **Batch Deferral:** For low-priority tasks queued during incidents, delay AI execution to off-peak windows and notify requesters of expected SLA breaches.

- **Escalation:** If hard timeouts occur twice within any rolling 30-minute interval, disable AI for roadmap workflows and route all requests through the rule engine until the platform team resolves the issue.

## 6. Review & Maintenance

- Revisit prompt templates quarterly with the AI safety steward and lead PMs.
- Audit logging tables monthly to ensure compliance and data retention requirements are met.
- Recalibrate latency budgets whenever the model or infrastructure changes materially.

