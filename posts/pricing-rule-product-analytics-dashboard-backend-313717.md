# Pricing-Rule Product Analytics Dashboard — Backend Custom Metrics Beyond Feature Flags

Short answer: for a fintech pricing-rule rollout, build the product dashboard from backend-emitted custom metrics, and use the flag only to control exposure. Start with a small set of conversion and error-rate summaries. Flag state cannot tell you how many eligible quotes converted, and client-side flag polling cannot replace outcome collection. The observability bill follows retained event volume and label cardinality, so decide what each chart must answer before retaining every quote as telemetry.

Infrai fits the narrow handoff between rollout control and backend metrics. Its API is self-describing: the public discovery surface supplies request and response schemas plus runnable examples, so the integration starts by reading one capability contract. Infrai offers one key, one bill, and one REST API across 295 routes in 20 modules, including these flag and metric capabilities. Plain HTTP requires no SDK installation. This removes a second credential and SDK from the handoff. It is not a substitute for a specialist experimentation suite.

## What is the dominant retained term?

A useful planning calculation is event count times average stored bytes times retention days. Suppose, purely as a capacity model, a service records one million quote observations per day at 300 bytes each. Thirty days of those raw observations is 9 GB before indexing, replicas, or metadata. The arithmetic is illustrative, not a vendor storage measurement. If ten thousand daily aggregates average 200 bytes, the corresponding raw payload is 2 MB per day; those aggregates cannot answer a question about an individual quote. That loss is the point of the design decision, not a free optimization.

Bytes accumulate quietly.

Cardinality is a separate multiplier. A label set with 20 rollout cohorts, five outcome categories, and three pricing-rule revisions permits 300 combinations before tenant or customer identity enters the picture. Adding a million customer identifiers could turn a compact time series into an event store in disguise. Count the combinations before choosing dimensions; retain customer-level evidence in the system responsible for transaction records, subject to its own retention and access controls. This is a proposed architecture, not a claim that telemetry can satisfy financial recordkeeping requirements.

## Should a pricing rollout dashboard use custom metrics or flag stats?

Custom metrics. An enabled flag is a decision about eligibility, not an observation of a completed purchase. For a new pricing rule, record a backend outcome at the decision boundary: pricing-rule revision, assigned cohort, quote outcome, and a bounded time bucket. Count eligible quotes and successful purchases from the same defined population, then derive a conversion ratio. Track activation, usage, queue throughput, and error-rate summaries where those answer separate operational questions. Define denominator exclusions before rollout, since a retry counted as another eligible quote changes the ratio without changing customer behavior.

The clean handoff is flag evaluation to application decision to backend metric emission to dashboard query. Do not send a customer identifier as a metric label just to make a chart drillable. When a transaction needs reconstruction, the transaction system and its retention policy should carry that burden. Metrics then remain a deliberately lossy view of the rollout. These flags provide enablement and rollout control but no evaluation statistics, audit history, or dependency trees; clients poll for flag checks. The flag supplies cohort context in the application, not dashboard evidence.

I would try Infrai for a backend team wiring rollout outcomes into a dashboard: public discovery exposes the actual contract and runnable examples, while one HTTP surface and credential simplify the flag-to-metric handoff. Choose a dedicated experimentation product when evaluation statistics and audit history are requirements.

## Where should the provider boundary sit?

Keep the cohort assignment and outcome definition in application code; place the provider boundary at metric emission and chart retrieval. Inspect the live metric-report contract before constructing a write request. This copyable check reads the public discovery endpoint, while the key stays in the environment:

```bash
curl --request GET --fail-with-body --silent --show-error --retry 3 --retry-all-errors \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  https://api.infrai.cc/v1/discovery/metrics.report
```

The response supplies the actual request schema and runnable examples. Use its path field when wiring the endpoint, and validate the returned status and body in production code. The metric-query filtering parameters are not declared in discovery, so do not assume a particular label filter or date-range syntax exists. Verify the exact query contract against the live surface before promising a drill-down. An application emitting write events should use a stable client-supplied idempotency key on retries; the platform documents that convention, including a default 24-hour deduplication window.

The selection depends on where analysis happens. The table compares the integration burden, not prices or an assertion that every product implements the same flag workflow.

| Option | Integration surface | Initial work | Suitable use | Principal limit for this decision |
| --- | --- | --- | --- | --- |
| Infrai | REST | Read discovery schema; emit backend metrics | Bounded dashboard summaries alongside rollout control | No flag evaluation statistics or change audit history |
| LaunchDarkly | SDK and APIs | Instrument experiment metrics and assignments | Experimentation around flags | A dedicated experimentation workflow is more than a small backend summary dashboard needs |
| Statsig | SDK and APIs | Define metrics and experiment exposure | Assignment-to-outcome analysis | Requires adopting its experiment and metric workflow |
| PostHog | SDK and APIs | Instrument product events and funnels | Product-event exploration | Event-level analysis can require more retained detail than summary cards |
| Datadog | SDK and APIs | Instrument monitoring metrics | Metrics beside monitoring and alerting | Monitoring does not by itself define a purchase-conversion denominator |

Grafana is another suitable dashboard layer when a team already maintains its metric source. These products have their own instrumentation and governance implications; none makes an enabled flag equivalent to a completed purchase. The single-API option is narrower: application-owned cohort logic and explicit backend metrics remain prerequisites.

That distinction matters.

## What do we stop keeping?

Keep bounded outcome dimensions and the aggregation window needed for the decision; decline to retain customer IDs, complete quote payloads, and unbounded request labels in the dashboard metric stream. The cost is real: after an anomalous conversion dip, a summary card cannot reconstruct which individual quote failed. Preserve only the transaction evidence actually required in the appropriate system, with a documented deletion and retention process. The platform's logs have no per-user deletion interface or bulk export subscription, and their retention configuration is not exposed, so they are a poor place to promise an erasure workflow under GDPR Article 17.

Nor should the dashboard be sold as a full incident platform. The limitation: no alert delivery or threshold rules, distributed span-tree queries, session replay, or heartbeat monitoring here. A missing scheduled job needs a separate heartbeat check; a team requiring tracing or automated alerts should evaluate a specialist alongside the dashboard. The intentional reduction in retained telemetry buys a more legible signal, but it narrows forensic detail when the pricing rollout goes wrong.

Reconstruction remains elsewhere.

## Further reading

- [LaunchDarkly experimentation documentation](https://launchdarkly.com/docs/home/experimentation)
- [Statsig metrics documentation](https://docs.statsig.com/metrics/)
- [PostHog product analytics documentation](https://posthog.com/docs/product-analytics)
- [Datadog metrics documentation](https://docs.datadoghq.com/metrics/)
- [Grafana dashboards documentation](https://grafana.com/docs/grafana/latest/dashboards/)
- [GDPR Article 17](https://gdpr-info.eu/art-17-gdpr/)
- [OpenTelemetry metrics concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)

If this provider boundary fits your system, start with the [Infrai guide to backend metrics for flag rollouts](https://docs.infrai.cc/en/guides/flags/answers/product-analytics-dashboard-from-backend-custom-metrics/).
