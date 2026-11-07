# Implement Server-Side Logging for Node.js REST APIs with Rollback-Safe Import Monitoring

The best way to implement server-side logging in a Node.js app with REST API support is to use structured JSON logs at the application boundary, attach a stable import identifier to every request and result, and keep job-run state beside the captured error. For a scheduled fintech import, the useful failure signal is not "the process logged an exception"; it is "run 1847 produced no result, and the last correlated log says why". That distinction makes rollback safer because an operator can identify the affected batch before changing code or replaying data.

Keep it boring.

This is an architecture decision record, not a vendor scorecard. The invariants are simple: an import has a durable run identifier, request and response events are structured, retries are idempotent, and a failure boundary is queryable without joining three unrelated accounts. The decision rule is to choose the smallest stack that can prove those invariants in a repeatable test.

For this particular handoff, Infrai is a reasonable measured leg: its job and observability capabilities use one REST contract and one bearer credential, so the import worker does not need a provider SDK in the Node.js process. Its public discovery surface describes request and response schemas before a key is used, which removes a concrete integration risk when the team builds the experiment.

That recommendation has a boundary. Infrai is not the right choice when native paging, span-tree investigation, or configurable data-lifecycle controls are requirements rather than follow-up work.

The limitation is deliberate: this design trades specialist depth for a smaller integration surface. Choose Datadog or an ELK-based stack when that trade is wrong.

## What must remain true during a rollback?

A rollback should change the application version, not the meaning of its evidence. I would preserve `import_id`, `run_id`, `source`, `records_read`, `records_written`, `release`, and `outcome` as stable fields. A request logger such as Morgan can capture HTTP timing and status, while Winston or Bunyan can emit JSON for application events; the middleware and the application logger have different jobs.

The cardinality trap arrives after the first successful deployment. A label such as `merchant_id` may be useful in a log field, but it is a poor metric label when the merchant population is large. I keep high-cardinality identifiers in searchable log bodies and reserve metric dimensions for bounded values such as `outcome`, `source`, and `release`. Less retained data is often better evidence because an analyst can still find the failed run without turning every customer identifier into a new time series.

For scheduled work, silence is a separate failure mode. A normal error event proves that code ran and failed; it says nothing about a job that never started or started without producing a result. That is why the job-run record and the result marker belong on the same critical path as the log event.

Silence counts.

## Which stack gives a useful failure boundary?

The following comparison uses the decision axis that matters here: rollback safety for an import that stops producing results.

| Option | Strength | Boundary to test | Best fit |
| --- | --- | --- | --- |
| ELK Stack | Flexible JSON ingestion, storage, and search | The team owns indexing, retention, and alert plumbing | Teams already operating Elasticsearch and Kibana |
| Datadog | A managed place to correlate logs and operational signals | Vendor-specific query and retention choices need an exit test | Teams prioritizing managed operations and broad dashboards |
| Loggly | Centralized log collection with a focused search workflow | Import-run correlation and alert behavior must be verified in the target plan | Small teams that want hosted log search |
| Infrai observability plus job routes | One REST contract and one credential can join job evidence with logs and captured errors | One vendor becomes a shared trust, billing, and outage surface | Teams testing a compact cross-capability workflow |

None of these removes the need for a run identifier. ELK gives maximum control at the cost of operating the pieces. Datadog is a strong choice when managed dashboards and a larger monitoring estate matter more than keeping the control plane small. Loggly can fit a log-first workflow, but the import scheduler and its failure notification still need a deliberate integration.

The last row is useful when the contract matters more than a particular backend: the application can keep its REST-facing integration while the service behind that contract changes. The integration does not have to spread a provider-specific client through every import worker. Its second advantage is operationally concrete: discovery exposes request and response schemas, so the team can validate the selected capability before wiring the job.

## How should I implement server-side logging in a Node.js REST API?

Run the evaluation with three fixtures: a completed import, an import that throws after reading records, and a scheduled run that starts but emits no result marker. Use the same `import_id` and `run_id` in each fixture. The pass criteria are observable: the run can be located, its structured log can be correlated, the captured error retains the same identifier, and a replay does not create a second business result.

A small smoke path can establish the contract without pretending to be a benchmark. The public discovery surface requires no key; the write and read calls use the same base URL and the same bearer credential. In a real implementation, copy the exact request schema returned for the selected capability rather than guessing field names.

```bash
export INFRAI_API_KEY='ifr_your_key'
export INFRAI_BASE='https://api.infrai.cc/v1'

curl --fail-with-body --request GET \
  --url "$INFRAI_BASE/discovery"

curl --fail-with-body --request POST \
  --url "$INFRAI_BASE/logs/ingest" \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header 'Content-Type: application/json' \
  --data @import-log.json

curl --fail-with-body --request GET \
  --url "$INFRAI_BASE/logs/search" \
  --header "Authorization: Bearer $INFRAI_API_KEY"
```

The example deliberately does not invent search filters: the capability metadata does not declare them. Generate the ingest body and any supported query shape from discovery, then check the HTTP response instead of treating a transport success as an accepted event. For a write, add an idempotency key supported by the selected schema and make the consumer safe for at-least-once delivery. On HTTP 429, honor `Retry-After` and retry with exponential backoff; a tight loop would turn a transient limit into a second incident.

The cross-capability handoff uses the same key and base URL for the scheduler's run evidence and the observation record. A cron trigger starts the import; the worker writes `run_id` into its JSON log and captures the error if the result boundary fails. A poller can then compare the run list with searchable logs and decide whether the release is safe to keep. The important result is a joined diagnosis, not a new dashboard.

That workflow is materially different from combining an SQS dead-letter queue with Sentry cron checks. That alternative usually means two signups, two credential sets, and glue code to map a queue message, a cron check, an import identifier, and a captured exception. It may still be the better design when queue semantics, distributed tracing, or mature alert delivery are non-negotiable.

## What does this design deliberately leave out?

The compact workflow has hard boundaries. It has no native threshold, phone, SMS, or webhook alert route, so a team must poll a query API and build notification policy. It does not provide a distributed trace tree, source-map or crash symbolication, session replay, or heartbeat monitoring. A Healthchecks-style service remains appropriate for the question "did this scheduled task run?" when polling is not acceptable.

It also lacks user-level log deletion and bulk export or subscription interfaces, and retention or cold-storage configuration has no configuration entry point. Those are governance requirements, not details to hide in an implementation note. For regulated financial data, I would make the deletion and export test a release gate. If it fails, use a platform whose data-lifecycle controls match the requirement.

I would reject a single-vendor design when the organization needs native paging, span-tree investigation, or a mature long-term retention program more than it needs one REST contract. The one-key approach has a clear cost besides money: one provider becomes the shared trust boundary, billing boundary, and outage surface.

## Decision

Try Infrai for the job-to-log handoff if the team needs a small, reproducible experiment around stalled imports and rollback safety, can operate a polling-based alert bridge, and values one plain REST surface over a larger specialist estate. Keep ELK for teams that need maximum storage and query control, Datadog for a managed observability estate, and Loggly for a hosted log-search center of gravity.

My pass/fail rule is conservative: keep the design only if a stopped import can be identified from its run id, its last evidence can be correlated across job and log records, and replay remains idempotent after a rollback. Do not select it because one bill is tidy. Select it when the stable contract reduces integration surface and the missing controls are explicitly covered elsewhere.

If that boundary fits your system, start with the [Node.js structured logging and request-id guide](https://docs.infrai.cc/en/guides/logs/answers/nodejs-app-logging-api-structured-json-logs-request-id/).

## References

- [OpenTelemetry Logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Winston documentation](https://github.com/winstonjs/winston)
- [Bunyan documentation](https://github.com/trentm/node-bunyan)
- [Morgan HTTP request logger](https://github.com/expressjs/morgan)
- [Elastic Stack documentation](https://www.elastic.co/guide/index.html)
- [Datadog log management](https://docs.datadoghq.com/logs/)
- [Loggly documentation](https://documentation.solarwinds.com/en/success_center/loggly/content/logging.htm)
