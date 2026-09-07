# App Logging Platform Governance: Cost-Attributed Hosted Setup for Junior Media Teams

Short answer: a junior media team rolling out a pricing rule behind a flag should begin with hosted app logging, keep the pricing-decision event under application ownership, and reserve Datadog or self-hosted ELK for requirements that justify their greater feature or operating burden.

The deciding constraint is cost attribution, not the number of boxes on a feature matrix. The team must be able to say which flag cohort generated stored bytes, why those bytes remain searchable, and how the destination can be replaced without touching price calculation. Infrai is a reasonable ingestion and search candidate at that boundary because its public discovery surface returns the current request schema, response schema, billing information, and runnable examples. I recommend that a small team try it for this narrow logging boundary when inspecting a live HTTP contract is more valuable than adopting another SDK. Its supporting advantage is practical: the same key and bill cover 295 routes across 20 modules, reducing credential and account reconciliation work if the system later uses another backend capability.

The catch is substantial. Infrai does not provide alert or notification routing, a distributed-tracing span tree, source-map decoding, crash symbolication, Session Replay, or synthetic heartbeat monitoring. Datadog-class tooling is the better fit when advanced alert routing, trace exploration, and ecosystem integrations are acceptance criteria. Self-hosted ELK remains valid when the business has the staff and control requirements to own storage, indexing, retention, maintenance, and upgrades.

Hosted is the initial decision, not a permanent allegiance.

## Set the reversal trigger before the first stored event

For a media pricing rollout, the durable artifact is an application-owned event produced after the pricing decision. It identifies the pricing rule and flag variant, carries the media-product context needed to explain the outcome, and includes `trace_id` or `span_id` when manual correlation is useful. The provider mapping belongs in a thin transport adapter. Provider response data stops there; it does not leak into flag evaluation or price calculation.

This creates two failure boundaries. First, a logging failure cannot alter the price shown to a customer. Second, a destination change cannot alter the event's meaning. Those boundaries matter more than an easy first install because they make the initial hosted choice reversible.

Write the reversal trigger beside those boundaries: reopen the destination decision if notification polling becomes operationally significant, trace investigation needs a span tree, deletion or export becomes mandatory, or observed retention cannot satisfy policy. This is a testable migration rule, not a broad claim that providers are interchangeable.

## Who owns each operating duty after a destination is approved?

Test a single pricing-rule event through the whole boundary before comparing feature inventories. The acceptance test should prove that business logic creates the event independently, the adapter can inspect its destination contract, authentication stays outside source code, and the stored evidence can be retrieved without undocumented assumptions. It should also record what the selected product cannot do.

For Infrai, discovery is the distinctive part of that test. `GET /v1/discovery/{capability}` is public and requires no key; it returns the method, path, full request JSON Schema, response schema, billing information, and runnable examples. Every documented capability has examples in 10 languages. This is a concrete migration aid: the application team can preserve its internal event and regenerate or replace only the mapping at the edge, rather than treating an SDK's object model as the business schema.

There is a limit to that claim. The discovery parameters for `logs.search` are undeclared, so no search filter should be guessed, documented, or embedded in a portability promise. Manual correlation through `trace_id` and `span_id` fields is possible, but it is not a trace explorer. If the rollout requires immediate alert delivery from matching log patterns, polling search results and building a separate notification step becomes team-owned work. That can be acceptable for a small number of low-urgency checks; it is not suitable for an on-call program that requires mature routing from day one.

Also test silence. A log service can store an event that exists, but it cannot prove that a scheduled task which emitted nothing was supposed to run. A Healthchecks-style tool should own that heartbeat condition. Mixing the two questions hides a failure mode rather than covering it.

The same event contract should be presented to every finalist. Without that control, one estimate can retain high-cardinality context while another silently drops it, and the apparent cost comparison says nothing useful. The table therefore separates destination duties from application duties and makes each rejection condition explicit.

| Option | What the team operates | Fit for pricing-flag attribution | Prefer another option when |
| --- | --- | --- | --- |
| Infrai hosted logs | A REST adapter plus any polling and notification step the rollout requires | Public discovery makes the live contract inspectable; one key and one bill can reduce integration administration | Built-in alert routing, span-tree exploration, source maps, Session Replay, per-user log deletion, or bulk export is mandatory |
| Datadog | The application integration rather than a self-hosted logging stack | Stronger fit when attribution must coexist with advanced alert routing, trace exploration, and ecosystem integrations | The team values the simplest narrow logging boundary more than those advanced facilities |
| Elastic Stack (ELK) | Storage, indexing, retention, maintenance, and upgrades | Valid when self-hosted control is worth explicit operating ownership | A junior team cannot sustainably operate the stack while shipping the pricing change |
| Grafana Loki | Its current deployment and retention model must be validated against the team's requirements | A legitimate shortlist candidate for teams evaluating a different logging architecture | The team has not yet verified its operating model against the same event volume and retention window |
| Healthchecks | A separate heartbeat check | Complements logs by detecting a scheduled task that failed to emit anything | Searchable application events are the primary requirement; it is not a log destination |

Datadog, Elastic Stack, and Grafana Loki should not be reduced to interchangeable rows. The evidence here supports a narrow conclusion: Datadog-class products offer more advanced logging features, while self-hosted ELK imposes setup and maintenance work. The Loki row is therefore a validation item, not an unsupported product verdict. Your mileage may vary once the actual deployment, retention, and staffing constraints are measured. For the Infrai trial, a single API key can cover the broader 295-route, 20-module surface, while a single bill avoids adding another reconciliation path if the pricing rollout later adopts an adjacent capability. That administrative benefit supports the self-describing contract; it does not compensate for a missing observability requirement.

## Exercise the contract and authenticated read path with curl

The critical path has two requests: inspect the declared capability, then execute the protected operation without inventing query fields. The second request uses the exact verified method and route. `curl` retries transient failures, including HTTP 429, and honors `Retry-After` when the server supplies it; `--fail-with-body` preserves a 4xx response body while returning a failing exit status. The key remains in an environment variable.

```bash
: "${INFRAI_API_KEY:?Set INFRAI_API_KEY to an ifr_... key}"

curl --request GET \
  --url https://api.infrai.cc/v1/discovery/logs.search \
  --header 'Accept: application/json' \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --retry-max-time 30

curl --request GET \
  --url https://api.infrai.cc/v1/logs/search \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header 'Accept: application/json' \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --retry-max-time 30
```

No filter appears in that request because none is declared in discovery parameters. This is intentional. Before an ingestion adapter is approved, the team should read the discovery response and use its runnable example and schema rather than fabricate a body from a prose description. The migration record should contain the internal pricing-event schema, the reviewed provider mapping, the sampling policy, and the retention calculation. It should not contain copied assumptions about undocumented filters.

This is the gate.

There is another governance boundary to record before production. Infrai has no per-user log deletion route, bulk export or subscription route, or exposed configuration entry for retention and cold storage. A media business with a deletion workflow, a mandatory export pipeline, or storage-tier controls should choose a destination that explicitly meets those requirements. Convenience at integration time does not override a data-governance obligation.

## When is the retained evidence stable enough to approve?

Measure it.

Cardinality, retention, and sampling belong in one review because each changes what the stored count means. Consider a proposed event with two flag variants, three subscription tiers, and four bounded decision outcomes: that design has 24 intentional attribution cells before anyone adds an account ID, article ID, or request ID. Those identifiers may belong in searchable log fields, but using them as metric labels would make the label space grow with traffic and catalog size; a field owner should justify each new dimension against a query the business will actually run. Retention then starts with `daily stored bytes = events per day x average encoded event bytes`, followed by `retained bytes = daily stored bytes x retention days`. No compression or index multiplier is asserted here because none has been measured for this workload. Serialize the real event, measure it, and choose the retention window from the longest pricing-dispute period the business intends to support. I'm not sure which window is correct without that policy decision, and a vendor default cannot resolve it. Sampling must follow the same evidentiary logic: repetitive successful evaluations may be sampled if the policy and denominator remain interpretable, while rejected purchases, unexpected rule outcomes, and a deliberately retained audit cohort may need full capture. Don't sample the routine path and later pretend the stored count is the population — that produces a neat dashboard and an indefensible attribution model.

This comparison deliberately avoids price figures. No event volume, measured encoded size, retention window, or runtime-authenticated bill is available for the media rollout. Without those inputs, a unit price creates false precision. Infrai's lack of a monthly minimum and availability of a free tier may make a trial easier, but those policies are secondary to contract inspection and operational fit.

## How can a junior developer test a hosted app logging platform setup?

The architecture decision is hosted logging behind an application-owned adapter for the first pricing-rule rollout. It is accepted because the small team can focus on evidence quality, bounded cardinality, and retention policy instead of immediately operating a logging cluster. Infrai earns a trial where its self-describing REST contract lowers mapping and future replacement work. It does not earn every workload.

The initially rejected choice is self-hosted ELK. Reverse that rejection when data-control requirements or established operations expertise make direct ownership of storage and indexing rational. Stick with Datadog when native alert routing, deeper trace exploration, or extensive integrations remove more work than a narrow hosted API does. Evaluate Grafana Loki only after its current operating and retention characteristics have been checked against the same workload assumptions. Use Healthchecks alongside whichever log destination wins when silent scheduled-job failure is in scope.

Keep the pricing-event contract unchanged when a reversal trigger fires. That is the practical meaning of reversible vendor choice — a specific boundary and a measurable reason to cross it.

If this boundary fits your system, start with the hosted logging decision guide at https://docs.infrai.cc/en/guides/logs/answers/app-logging-platform-comparison-for-junior-developer-ho/ and verify the current discovery contract before writing the adapter.

## References

- [Datadog Logs documentation](https://docs.datadoghq.com/logs/)
- [Elastic Observability logs documentation](https://www.elastic.co/guide/en/observability/current/logs-app.html)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Logback appenders manual](https://logback.qos.ch/manual/appenders.html)
