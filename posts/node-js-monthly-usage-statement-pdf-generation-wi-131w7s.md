# Node.js Monthly Usage Statement PDF Generation with Scheduled Customer Email Delivery

The least complex reliable design is a frozen monthly usage ledger, one PDF derived from that ledger, and a separate delivery record for every game-studio account. **Short answer:** schedule only the discovery of accounts in Node.js. Give each statement a stable period key, calculate it from immutable meter entries, retain the compact evidence needed to reproduce the total, and let rendering and email delivery retry independently.

This avoids paying to keep every operational event merely because an invoice may be challenged later. It also makes the important question answerable: which authorized process read which usage rows, produced which bytes, and addressed which recipient?

The bill for this system is mostly repeated data volume: raw gameplay telemetry retained for months, high-cardinality labels copied into logs, PDF artifacts, and duplicate render or mail attempts. The PDF itself is usually the wrong first target. A small ledger plus explicit access evidence moves the dominant retention term because invoice support no longer depends on the full event stream.

## How should Node.js generate and email each monthly usage statement PDF?

Schedule a coordinator that creates work; do not put aggregation, PDF rendering, and email in one long callback. At the start of a billing period, the coordinator selects eligible customer accounts and attempts to create one statement row per tuple of `(customer_id, period_start, period_end, revision)`. A uniqueness constraint on that tuple turns a repeated scheduler tick into a lookup instead of a second invoice.

The statement worker then reads a closed interval according to one documented boundary convention, computes meter totals, and stores the resulting ledger snapshot. For a gaming platform, useful dimensions might be game title, region, meter name, quantity, and unit. Do not carry player IDs, session IDs, request IDs, or match IDs into the monthly statement unless the contract requires them. Those fields multiply cardinality while contributing nothing to a studio's invoice total.

Keep that boundary small.

Next, a renderer consumes the frozen snapshot and writes a PDF artifact with its content digest. A mail worker receives only the statement identifier, loads the approved recipient at send time, and records the delivery attempt. Rendering can retry without sending. Sending can retry without recalculating. A correction creates a new revision rather than mutating the evidence beneath an already delivered document.

One internal request can enqueue the period safely. The endpoint here is illustrative; its authentication and authorization belong at the service boundary.

```bash
curl --fail-with-body \
  --request POST \
  --header "Authorization: Bearer ${BILLING_JOB_TOKEN}" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: statements-2026-08" \
  --data '{"period_start":"2026-08-01T00:00:00Z","period_end":"2026-09-01T00:00:00Z"}' \
  https://billing.internal.example/jobs/monthly-statements
```

The token should come from a secrets-management system, not source code, a copied command history, or the scheduler definition. The OWASP Secrets Management Cheat Sheet describes centralized storage, access control, rotation, expiration, and auditing as parts of the secret lifecycle. In this design, the scheduler credential is permitted to enqueue a period; it is not a general database or mail credential.

## Count bytes and cardinality before choosing retention

Retention math should be done on the records that dominate storage, not on the tidy statement table. Let `E` be raw events per month, `B` their average stored bytes after indexing and replication are accounted for, and `R` the number of retained months. The raw-event footprint is `E x B x R`. If logs repeat `customer_id`, `player_id`, `session_id`, and `request_id` on every processing step, their index and label cardinality become a second, less obvious retention surface.

Use a concrete planning case, clearly labeled as an example rather than a benchmark. Suppose the platform receives 180 million billable events per month and an internally measured stored footprint averages 420 bytes per event. Keeping twelve months represents about 907.2 billion bytes before backups. A monthly ledger containing 40,000 customer-and-meter rows at 300 bytes each is about 12 million bytes per month. These numbers are arithmetic inputs, not universal performance claims; teams must substitute measurements from their own storage path.

| Data class | Purpose | Retention decision | Cardinality control |
| --- | --- | --- | --- |
| Raw meter entry | Recompute and investigate recent usage | Keep for the contractual dispute window | Partition by time; avoid telemetry labels derived from player or session IDs |
| Monthly ledger row | Reproduce billed totals | Keep for the required financial and audit period | Aggregate to customer, title, region, meter, and period |
| PDF plus digest | Show exactly what was delivered | Keep with the statement record | One artifact per statement revision |
| Access event | Prove reads, renders, approvals, and sends | Keep long enough to cover statement audits | Record actor, action, object, outcome, and time; omit payload copies |
| Debug trace | Diagnose processing defects | Sample and expire quickly | Sample errors deliberately; do not promote trace IDs to unbounded metric labels |

The biggest change is deliberate: aggregate raw events into a frozen ledger as soon as the period and late-arrival policy allow, then expire raw events when the dispute and compliance rules permit. Keeping twelve copies of essentially the same dimensions in events, logs, traces, and exports does not increase invoice correctness twelvefold.

Sampling requires a split decision. Never sample the billable meter entries that feed the ledger. Diagnostic traces may be sampled, while error outcomes and state transitions should be retained at a higher rate according to a documented policy. Metrics need bounded dimensions. Otherwise, per-player or per-session labels convert a useful counter into an expensive index whose growth follows the player population.

No sample can repair a missing invoice fact.

## How does the ledger make access auditable?

Auditability is a chain of identities and hashes, not a large log pile. Each transition should record the authenticated actor or workload, action, statement ID, prior state, resulting state, timestamp, and outcome. Record the ledger digest when aggregation closes, the artifact digest after rendering, the recipient identity or stable internal reference at approval, and the provider-neutral delivery message ID after acceptance.

Do not log the PDF body, mail body, access token, or complete customer profile. Those copies increase both exposure and retention cost. The access record needs to prove that an authorized principal performed a defined action on a defined object. It does not need to clone the object.

**Separate authority by stage.** The coordinator may enumerate billable accounts and enqueue IDs. The aggregator may read meter entries and write ledger snapshots. The renderer may read snapshots and write artifacts. The mailer may read approved artifacts and recipient data. No single workload needs unrestricted access to raw telemetry, customer contacts, artifacts, and sending credentials.

A compact statement record can carry the control points:

```bash
curl --fail-with-body \
  --request POST \
  --header "Authorization: Bearer ${STATEMENT_WORKER_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{"statement_id":"stmt_7f31","expected_revision":1,"ledger_digest":"sha256:8d3f...","action":"request_render"}' \
  https://billing.internal.example/statement-transitions
```

The abbreviated digest is suitable only for exposition. A real request sends the complete digest produced over a canonical ledger representation. Define that representation before implementation: field order, numeric precision, time zone, and handling of missing dimensions all affect the bytes and therefore the digest.

## Failure boundaries matter more than the PDF library

Month boundaries are a common source of quiet billing errors. Store instants in UTC, use a half-open interval such as `period_start <= occurred_at < period_end`, and define which business time zone selects those bounds. Late events need an explicit rule: hold closure for a grace period, apply them to the next statement, or issue a revision. The contract decides; the worker must not improvise.

The database transaction should close the ledger and enqueue an outbox record together. A dispatcher can then publish render work repeatedly while the consumer deduplicates by statement and revision. This pattern narrows the crash window between changing durable state and asking another process to act. It also gives operators a finite set of states to reconcile: discovered, aggregated, rendered, approved, accepted for delivery, or failed.

Delivery acceptance is not proof that a person read the message. Keep those meanings distinct. Store the mail system's acceptance identifier and later delivery events when available, but do not rewrite history by labeling acceptance as delivery. Retries must reuse a stable delivery key. Before any send, compare the current statement state and artifact digest with the approved values.

This architecture has limitations. It is a poor fit when a customer contract requires an interactive, continuously changing statement rather than a period-close document, or when every historic gameplay event must remain available indefinitely for a separate legal purpose. In the first case, use a query-backed account view and produce a PDF only as an export; in the second, place the mandated archive behind distinct access controls and retention governance instead of pretending the compact ledger replaces it. The staged pipeline also costs more operational effort than a single process: it needs durable state, idempotency, reconciliation, and workers that can coexist across deployments. For a small game with a few manually reviewed accounts, a controlled monthly batch may be the more proportionate choice. Move to queued stages when retry isolation and an auditable chain are worth that complexity.

The boundary is contractual.

PDF rendering deserves deterministic inputs. Pin the renderer version, fonts, locale, and page template; load no remote assets during a job. Test page overflow with long game titles, large quantities, missing optional dimensions, and enough rows to cross several pages. A snapshot test can detect visual changes, while a text extraction test checks that totals and identifiers survived rendering. Neither test substitutes for recomputing the ledger from known fixtures.

Deploy schema changes in compatible steps. Add new fields as optional, teach workers to read both representations, backfill when needed, and only then make the new representation mandatory. Queue workers from two releases may overlap during a rolling deployment, so a job payload should remain small and versioned.

## Operate the pipeline without storing the pipeline again

The useful operational counters are bounded: statements discovered, aggregation successes and failures, render attempts, send attempts, age of the oldest unfinished statement, and counts by finite state. Customer IDs belong in searchable audit records with controlled access, not in metric labels. The same rule applies to statement IDs and delivery IDs.

Logs should explain state transitions with a compact schema. Retain all failed transitions for the investigation window, but sample repetitive success logs once aggregate counters and audit events prove progress. This is a trade: aggressive sampling weakens sequence-level debugging, while unsampled success logs charge storage for thousands of identical confirmations. Preserve the durable state machine and access trail; sample the narration around them.

Before the first monthly run, test duplicate scheduler ticks, worker termination after each durable write, renderer timeouts, rejected recipients, credential rotation, and a correction after delivery. Reconciliation should compare expected eligible accounts with terminal statement states and surface gaps without scanning every raw gameplay event.

The deliberate loss is clear. After raw-event expiry, an engineer cannot replay an old invoice from every original gameplay fact or inspect a discarded trace for one player's session. The retained ledger, calculation version, digests, artifact, and access events can establish what was billed and delivered, but they cannot answer every later product-debugging question. **That loss is the price of bounded retention.** Choose the raw-data window from contractual, regulatory, and investigation needs, document it, and resist extending it merely because deletion feels uncomfortable.

## Further reading

- OWASP, "Secrets Management Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- OWASP, "Logging Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- IETF, "The OAuth 2.0 Authorization Framework" (RFC 6749): https://www.rfc-editor.org/rfc/rfc6749
- PostgreSQL documentation, "Date/Time Types": https://www.postgresql.org/docs/current/datatype-datetime.html
- Transactional Outbox pattern: https://microservices.io/patterns/data/transactional-outbox.html
