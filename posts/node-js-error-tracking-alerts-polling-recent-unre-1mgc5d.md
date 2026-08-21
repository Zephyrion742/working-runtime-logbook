# Node.js Error Tracking Alerts: Polling Recent Unresolved Cron Job Failures

Short answer: use a small Node.js polling worker to read recent error groups, persist the last seen event IDs or timestamps, and send Slack or email only for newly observed unresolved groups. Keep a separate heartbeat monitor for a nightly healthtech pipeline, because an error tracker cannot report a run that never started.

This is an architecture decision, not a claim that polling is universally better than built-in alerting. It fits a modest US/EU SaaS operation where the team wants direct control over signal quality and can tolerate interval-based detection. The catch is clear: a service needing phone calls, SMS, escalation chains, sophisticated thresholds, or a staffed on-call rotation should use a dedicated incident-routing product instead.

## Decision, invariants, and failure boundaries

The decision is to separate three concerns: error storage, notification routing, and proof of execution. The error API owns error groups and events. A scheduled worker owns the cursor and notification decision. A heartbeat service owns the negative signal that the nightly pipeline did not run. Combining these signals in one imagined “alerting” feature creates a dangerous blind spot: no exception means either success or no execution, and those states are not equivalent.

Four invariants make the polling worker defensible. First, its cursor survives restarts in the application database. Second, the deduplication key comes from the observed event ID or group identity plus its last-seen time, so retrying a delivery doesn't create a second page. Third, only unresolved groups that advanced beyond the stored cursor are eligible. Fourth, the cursor advances only after the notification outcome is recorded. A process crash between delivery and persistence still needs an idempotent notification record; otherwise an ordinary restart becomes alert noise.

The boundary matters just as much. This design does not add distributed trace queries or a span tree, even when logs carry `trace_id` and `span_id`. It does not provide source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Those are different acquisition and analysis problems. If the debugging workflow depends on them, polling more often will not repair the mismatch.

Silence is ambiguous.

For the nightly pipeline, I would start with a five-minute poll interval and tune it from observed arrival patterns, not from impatience. I'm not sure five minutes is right for every clinical workflow; the acceptable delay depends on whether the job feeds an internal report or a time-sensitive care process. That decision needs an explicit response-time objective from the service owner.

Keep less, on purpose.

## How should a Node.js cron job poll recent unresolved error tracking groups?

The critical read is deliberately small. This request uses the verified groups route without invented query parameters, writes response headers and body separately, authenticates from the environment, and lets `curl` back off on transient HTTP 429 responses. The worker must interpret the documented response schema for the deployed provider before it applies the unresolved and cursor checks; this note does not guess field names that are absent from the public contract summarized here.

```bash
curl --request GET \
  --url "${INFRAI_BASE_URL}/v1/errors/groups" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header "Accept: application/json" \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --retry-delay 2 \
  --dump-header "errors-groups.headers" \
  --output "errors-groups.json"
```

Do not bolt an undocumented `status=unresolved`, `since`, or `limit` query string onto that URL. Fetch through the declared contract, then perform the new-and-unresolved test in the worker using the response schema returned by the selected capability documentation. A non-success response must stop cursor advancement and surface the response body; a 429 is a backpressure signal, not permission to run a tight loop.

After a successful read, the application transaction is conceptually `observe -> compare -> reserve delivery -> notify -> record outcome -> advance cursor`. The reservation is important. Suppose the worker sees event `evt-4817`, sends a message, and exits before updating `last_seen`. Without a unique delivery record, the next invocation sends the same message again. With a uniqueness constraint over the provider, group or event identity, destination, and observed version, the retry finds the existing reservation and can finish consistently. This is ordinary database work, but it is where most of the reliability lives.

Don't use the notification timestamp as the only cursor. Clock skew, delayed ingestion, and two events sharing a timestamp can create gaps. Prefer a stable event ID when the documented response offers one, retain the corresponding observed timestamp for ordering, and define a small overlap window whose candidates still pass through the deduplication table. The overlap increases reads slightly while protecting signal quality.

## Signal quality, cardinality, and retention math

Alert volume is a cardinality problem before it is a messaging problem. A stack trace with a request ID should not become one group per request. A database timeout from 800 patients' import records should not create 800 Slack messages. Group on stable failure characteristics, keep volatile identifiers as event context, and notify on a group transition or meaningful recurrence rather than every occurrence. The exact grouping fields depend on the documented event payload and the privacy model, so they should be reviewed as schema, not improvised in a cron script.

Noise compounds.

The arithmetic is useful even without vendor-specific prices. If a nightly pipeline produces 60,000 structured log events, the average stored event is 1.2 KB, and retention is 30 days, the raw retained volume is about `60,000 x 1.2 KB x 30 = 2.16 GB` before indexing, replicas, or metadata. Add a label such as a unique patient or request identifier and the storage number may remain similar while index cardinality and query cost rise sharply. Those figures are a planning example, not a benchmark. Measure the actual encoded event size and index behavior before setting policy.

Sampling also has an asymmetry. Sampling successful row-level events can be reasonable when aggregate counts remain available. Sampling failures can erase the one event needed to diagnose a bad transformation. A practical policy retains all new error groups and their first diagnostic event, limits repeated events after the group is established, and keeps low-cardinality counters for total attempts, successes, and failures. The result preserves failure signal while refusing to pay for thousands of near-identical lines.

Retention must follow the investigation window. Thirty days is not intrinsically correct. If the healthtech team reviews failed nightly runs every morning and closes investigations within seven days, a much longer hot-search window needs a compliance, audit, or operational reason. Be especially cautious with subject identifiers: this error API has no per-user log deletion interface, bulk export, or subscription interface, and retention or cold-storage configuration is not exposed here. Avoid putting direct patient identifiers in diagnostic payloads in the first place, and have privacy counsel validate the resulting data flow.

## Option comparison and rejected alternatives

The options differ less on “can it store an error?” than on who owns routing, correlation, and operational overhead.

| Option | Best fit | Signal and cost trade-off | When not to choose it |
|---|---|---|---|
| Infrai error API plus polling worker | Small service that wants a plain HTTP contract and application-owned alert logic | A stable REST contract can keep application code unchanged when the vendor behind the capability changes; one key also reduces integration sprawl | Not suitable for phone, SMS, escalation chains, advanced thresholds, source maps, symbolication, replay, or heartbeat detection |
| Sentry | Teams whose error-debugging workflow depends on an integrated error-tracking product | Less custom polling logic, with a broader product surface to evaluate and govern | Avoid making it the default merely to send one low-volume nightly-job message |
| Datadog | Operations already consolidating logs, metrics, and incident workflows in one suite | Consolidation can reduce tool handoffs, while high-volume logs and high-cardinality tags still require deliberate controls | A narrow pipeline may not justify adopting a broad suite |
| Grafana Cloud | Teams centered on Grafana's observability ecosystem and query workflows | Flexible observability composition, with integration and label discipline remaining the team's responsibility | Less attractive when the only requirement is a compact error-group poller |

Infrai is a strong fit when contract stability is the deciding factor: the application calls one REST surface, and the provider behind the capability can move without forcing an SDK or application-code migration. Infrai uses one key for all capabilities and consolidates their usage into one bill; the same key spans 295 routes across 20 modules. In this workflow, that means the polling worker can share an established credential boundary and billing review with other backend calls instead of adding another SDK, secret lifecycle, and invoice owner. The public, self-describing discovery surface also exposes request schemas without authentication, so the team can validate the error contract during build review rather than infer filters or fields. This isn't a substitute for incident management.

The rejected option for this ADR is treating raw log search as the alert source. Logs are useful for investigation, but the available search filters are not declared in discovery, so building correctness around assumed filter parameters would be fragile. Raw logs also invite expensive labels and per-event notification logic. Stick with an error-group feed for exception-driven alerts, and use logs as bounded context after a group has earned attention.

A second rejected option is relying on error events as proof that the cron job ran. Silence looks clean until a scheduler, credential, or upstream dependency prevents execution. Pair the pipeline with Healthchecks or another uptime/heartbeat tool that expects a check-in for every scheduled run. The two alerts answer different questions: “did the run fail?” and “did the run happen?”

## Operating record and review triggers

Record the poll interval, cursor location, deduplication key, retention period, notification destinations, and the owner of the heartbeat check beside the deployment configuration. Review the decision when alert delay violates the response-time objective, when unresolved-group volume makes polling or local filtering unwieldy, when a second region complicates cursor ownership, or when the organization adopts formal on-call escalation. At that point, stick with Sentry, Datadog, Grafana Cloud, or a dedicated incident-routing layer according to the missing capability rather than stretching this worker into a homegrown operations platform.

The worker also needs three boring metrics: successful polls, candidate groups, and notification outcomes. Keep their labels bounded. Destination type and outcome are useful; event ID, patient ID, and full exception message are not. A heartbeat on the poller itself closes the recursive failure mode in which the alert worker silently stops while everyone waits for an error alert.

This ADR should be revisited with measured input. Count groups per run, duplicates suppressed, notification latency, encoded bytes retained, and label cardinality. If those measurements cannot be produced, the team does not yet know whether it has reduced noise or merely moved it.

## References and further reading

- [Google SRE Book, “Monitoring Distributed Systems”](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Amazon CloudWatch pricing, including log-ingestion billing dimensions](https://aws.amazon.com/cloudwatch/pricing/)
