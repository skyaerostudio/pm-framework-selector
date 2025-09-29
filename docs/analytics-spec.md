# Analytics Specification

## 1. Metrics Schema

| Metric Group | Metric Name | Description | Collection Method | Dimensions | Sampling | Retention |
| --- | --- | --- | --- | --- | --- | --- |
| Time to Interactive (TTI) | `tti_ms` | Milliseconds from navigation start until UI is reliably interactive. | Frontend emits via PerformanceObserver using `PerformanceEventTiming` or computed from `domContentLoadedEventEnd` + long-task heuristics. | `app_version`, `page_id`, `user_role`, `device_type`, `connection_type`, `experiment_variant` | 100% on production, 10% in staging. | 13 months (aligns with product analytics policy). |
| Time to Interactive | `tti_outlier_flag` | Boolean flag when `tti_ms` exceeds alert thresholds (e.g., > 5000 ms). | Derived in analytics pipeline during ingestion. | Same as `tti_ms` | 100% | 13 months |
| Cache Performance | `cache_hit_latency_ms` | Milliseconds between request dispatch and cache hit acknowledgement. | Backend instrumentation around cache middleware logging high-resolution timestamps. | `service`, `endpoint`, `cache_layer`, `hit_or_miss`, `region` | 100% | 25 months |
| Cache Performance | `cache_hit_rate` | Ratio of cache hits to total eligible requests per interval. | Calculated in data warehouse from raw hit/miss counters. | `service`, `endpoint`, `region` | 100% | 25 months |
| Completion Funnel | `funnel_stage` | Stage identifier for completion funnel (e.g., `viewed_form`, `started_form`, `submitted`, `confirmed`). | Frontend emits funnel steps via analytics SDK; backend confirms `confirmed`. | `user_id` (hashed), `session_id`, `experiment_variant`, `geo`, `device_type` | 100% | 25 months |
| Completion Funnel | `funnel_transition_latency_ms` | Time between consecutive funnel steps per session. | Derived in warehouse using event timestamps. | Same as `funnel_stage` | 100% | 25 months |
| Confidence Score | `confidence_score_value` | Float [0,1] representing framework recommendation confidence. | Backend scoring service includes in response payload; frontend forwards to analytics SDK. | `framework_id`, `candidate_set_size`, `model_version`, `user_role`, `experiment_variant` | 100% | 13 months |
| Confidence Score | `confidence_bucket` | Categorized confidence (`low <0.4`, `medium 0.4-0.7`, `high >0.7`). | Derived during ingestion for cohort analysis. | Same as `confidence_score_value` | 100% | 13 months |

### Event Definitions

1. **`framework_recommendation_viewed`**
   - Trigger: Page load when recommendation UI becomes visible.
   - Payload: `tti_ms`, navigation metadata, `confidence_score_value` (if already available), `app_version`.
   - Notes: Use Performance API to capture TTI just before dispatch.

2. **`framework_recommendation_interacted`**
   - Trigger: User first interacts (clicks/scrolls) with recommendation components.
   - Payload: `session_id`, `funnel_stage="started_form"`, timestamps.
   - Notes: Ensure duplicate suppression per session.

3. **`framework_recommendation_submitted`**
   - Trigger: Submit action initiated from frontend.
   - Payload: `funnel_stage="submitted"`, client timestamp, `confidence_score_value`.
   - Notes: Include validation result codes when available.

4. **`framework_recommendation_confirmed`**
   - Trigger: Backend confirms submission success.
   - Payload: `funnel_stage="confirmed"`, server timestamp, `confidence_bucket`.
   - Notes: Use server-timing header to relay cache hit info when applicable.

5. **`cache_lookup`**
   - Trigger: Backend cache middleware intercepts eligible request.
   - Payload: `cache_layer`, `hit_or_miss`, `cache_hit_latency_ms`, request identifiers.
   - Notes: Log to structured log sink and forward via analytics pipeline.

## 2. Tooling and Integration Points

### Frontend

- **Performance API & Long Task Observer**: Capture TTI by observing `PerformanceEventTiming` and `longtask` entries. Integrate within `src/frontend/metrics.ts` initialization hook.
- **Analytics SDK (Segment or RudderStack)**: Batch events `framework_recommendation_*` with schema-compliant payloads. Hook into existing `src/frontend/analytics.ts` dispatcher.
- **Server-Timing Header Reader**: Extend fetch client (`src/frontend/api/client.ts`) to parse `Server-Timing` for cache hit timing and propagate to analytics events.

### Backend

- **Cache Middleware Instrumentation**: Wrap cache access in `src/backend/cache/middleware.ts` to timestamp before and after lookup, emitting `cache_lookup` events and populating `Server-Timing: cache;dur=<ms>;desc="hit"`.
- **Recommendation Service**: Update `src/backend/services/recommendation.ts` to attach `confidence_score_value` in responses and log to analytics queue.
- **Analytics Pipeline**: Use Kafka topic `analytics.events` feeding into dbt models. Define schemas in `analytics/schemas/framework_metrics.yml` and enforce via schema registry.
- **Warehouse Transformations**: dbt models calculate `cache_hit_rate`, funnel transitions, and derived fields (`tti_outlier_flag`, `confidence_bucket`).

### Data Governance & Validation

- Enforce schema via analytics contract tests (Great Expectations) running nightly.
- Automated unit tests in CI ensure frontend emits required properties using Jest snapshots.
- Backend instrumentation covered by integration tests verifying presence of headers and event payloads.

## 3. Monitoring & Alerts (FR-007, FR-008, FR-010)

| Requirement | KPI | Threshold | Alert Channel | Dashboard |
| --- | --- | --- | --- | --- |
| FR-007 (Performance) | P95 `tti_ms` | > 4000 ms for 2 consecutive 5-min intervals. | PagerDuty (SRE Primary), Slack `#perf-alerts`. | Looker dashboard "Framework UX Performance" with TTI percentile tiles and region breakdowns. |
| FR-007 | `tti_outlier_flag` rate | > 5% of sessions daily. | Slack `#perf-alerts`. | Same dashboard, anomaly chart for outlier rate. |
| FR-008 (Cache Efficiency) | Cache hit rate | < 85% over 1 hour. | PagerDuty (Platform On-call). | Looker dashboard "Framework Cache Health" with hit/miss trend and latency heatmap. |
| FR-008 | `cache_hit_latency_ms` P90 | > 50 ms over 15 min. | Slack `#platform-alerts`. | Cache Health dashboard with latency percentile panel. |
| FR-010 (Recommendation Quality) | Average `confidence_score_value` | < 0.65 daily. | Product Ops email + Slack `#recommendations`. | Looker dashboard "Recommendation Confidence" with trendline and distribution histogram. |
| FR-010 | Funnel completion rate (`confirmed`/`viewed`) | < 45% daily. | PagerDuty (Growth On-call) if persists 2 days. | Recommendation Confidence dashboard with funnel conversion chart. |

### Alerting Implementation Notes

- Alerts backed by Looker data actions triggering PagerDuty/Slack.
- CloudWatch metrics for cache middleware emit near-real-time signals feeding SNS to PagerDuty for FR-008.
- Establish runbooks linked in alert descriptions covering mitigation steps.
- Weekly review of dashboards in product analytics meeting to ensure compliance trends.

### Dashboard Components

1. **Framework UX Performance**
   - Tiles: TTI percentile trend, outlier rate vs threshold, breakdown by device type.
   - Filters: `app_version`, `region`, `experiment_variant`.
   - Drill-down: Table of slowest sessions with context (network type, device).

2. **Framework Cache Health**
   - Visuals: Hit rate gauge, latency heatmap by region, top endpoints with misses.
   - Alerts overlay showing when thresholds breached.

3. **Recommendation Confidence**
   - Visuals: Confidence distribution, conversion funnel, correlation between confidence bucket and completion.
   - Annotations for model deployments (`model_version`).

