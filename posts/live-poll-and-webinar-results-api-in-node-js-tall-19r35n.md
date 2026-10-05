# Live Poll and Webinar Results API in Node.js: Tallying for Thousands of Viewers

Short answer: for a live poll or webinar results API in Node.js, aggregate votes in the server-side database, then publish a periodic tally to thousands of viewers. Broadcasting every individual vote turns fan-out into a participant-squared problem; a tally keeps the channel carrying the view while the database remains authoritative.

For an edtech editor with a collaborative cursor and a live poll on the same page, this distinction matters. A poll with 10,000 viewers does not need 10,000 copies of every click. It needs a result that is fresh enough to be useful, plus a trustworthy count if the connection drops.

## What should a Node.js live poll and webinar results API publish?

Publish state, not the event stream. Each vote can be validated and counted by the Node.js service. The service stores the authoritative total, grouped by option and poll version, then emits a snapshot such as “option A: 4,821; option B: 3,907” on an interval. Viewers render that snapshot and do not become part of the counting system.

The interval is a product decision. For a fast audience pulse, one update every second may feel immediate. For a long webinar, five seconds can be adequate and produces one fifth as many result messages. I would choose the slowest interval that still supports the host's decision, then document it as a freshness target rather than imply that every click is visible instantly.

There is a small but important ordering rule: assign a monotonically increasing poll revision when the tally is published. Clients ignore an older revision that arrives after a newer one. This handles reconnects and ordinary network reordering without pretending the transport is a transaction log.

That is the whole trick.

Keep the cursor path separate. Cursor updates are ephemeral and user-specific; poll totals are shared state. They may use the same realtime provider, but they should not share retention, authorization scope, or payload assumptions.

## Where does the telemetry and retention bill actually come from?

The dominant term is fan-out. If every one of N participants receives every one of N vote events, delivery work grows approximately with N squared. At 10,000 participants, that is the shape of 100 million deliveries before retries, reconnects, or cursor traffic. A periodic tally changes the result stream to N multiplied by the number of intervals, while vote validation and aggregation stay on the write side.

That math is why I count bytes and labels before selecting a vendor. A payload containing a poll id, option id, revision, and counts is cheaper to store and easier to inspect than a verbose event with user profile fields. Do not attach a high-cardinality label such as `user_id` to every result metric. Keep that identifier in an audit record when policy requires it; leave it out of the hot telemetry path.

Retention should follow the question you will ask later. Keep the current tally and a compact audit trail for reconciliation. Sample cursor telemetry, and expire raw cursor points quickly. The catch is that deliberate deletion removes forensic detail: after an incident, you may know that a revision was late without knowing every cursor position that preceded it. That is an acceptable trade when the poll result, rather than cursor replay, is the business record.

Measure three quantities separately: votes accepted, tally snapshots published, and snapshot deliveries. Their units differ. Mixing them into one “events” counter makes a quiet system look cheap and a busy one look mysterious.

For a concrete budget exercise, assume a 45-minute webinar, 10,000 viewers, and a five-second tally interval. The result channel carries 540 snapshots per viewer, or 5.4 million viewer deliveries, while the write path handles only the votes that people actually submit. If the host changes the interval to one second, the audience sees fresher numbers but the delivery term becomes five times larger; no dashboard can hide that multiplication. I would record the interval beside the metric, because “5.4 million messages” without a time window is not an actionable observation. The database still stores one authoritative revision per accepted update, so a reconnect can be served from state rather than replaying 45 minutes of events. Those are planning figures, not a benchmark, and I am not claiming a provider will deliver them at a particular latency.

## How can thousands of viewers stay in sync without trusting clients?

Treat the browser as an untrusted proposer. It can submit a vote intent with a poll id and option, but the server decides whether the token is allowed to vote, whether the poll is open, and whether a duplicate submission is ignored. The database transaction increments the count and records the decision; publication happens after the committed revision exists.

For reconnects, send the latest committed snapshot immediately, then resume the interval. Do not ask a newly connected viewer to replay every vote. If the viewer needs proof of participation, return a receipt from the write path; the shared result channel should stay free of personal data.

Token scope is the primary trust boundary. Give a viewer a token that can read the poll's result channel, not one that can publish arbitrary events or edit the poll. Rotate scopes per session or room, and reject a token whose poll id does not match the request. Your mileage may vary on the exact expiry window; choose it from the longest expected webinar segment plus a reconnect margin, then test expiry during a live rehearsal.

The publication request should be retried with an idempotency key derived from the poll revision. A timeout does not tell the client whether the provider accepted the message. Idempotency makes that uncertainty survivable; it does not make a failed database transaction disappear. Alert on a revision that is committed but not published, and let a worker republish the same revision.

Here is the shape of a minimal publisher. The payload fields represent the already-committed view; the database write happens before this command. The loop honors `Retry-After`, surfaces non-success responses, and gives revision 42 a stable idempotency key.

```bash
api="https://${INFRAI_HOST}/v1/realtime/publish"
body='{"channel":"poll-results-42","event":"tally","data":{"revision":42,"counts":{"a":4821,"b":3907}}}'

for attempt in 1 2 3 4; do
  response=$(curl --silent --show-error --write-out '\n%{http_code}' \
    --request POST "$api" \
    --header "Authorization: Bearer $INFRAI_API_KEY" \
    --header 'Content-Type: application/json' \
    --header 'Idempotency-Key: poll-42-revision-42' \
    --data "$body")
  status=${response##*$'\n'}
  payload=${response%$'\n'*}
  if [ "$status" -ge 200 ] && [ "$status" -lt 300 ]; then
    printf '%s\n' "$payload"
    break
  fi
  if [ "$status" -ne 429 ] || [ "$attempt" -eq 4 ]; then
    printf 'publish failed (%s): %s\n' "$status" "$payload" >&2
    exit 1
  fi
  sleep_seconds=$((2 ** (attempt - 1)))
  sleep "$sleep_seconds"
done
```

## Which realtime option fits this live poll architecture?

The vendors below can all carry a published view. Their differences are about the surrounding operating model, not a license to skip server-side aggregation.

| Option | Useful fit | Trade-off to check |
| --- | --- | --- |
| Ably | Managed pub/sub with channels and presence-oriented features | You still design tally storage, authorization, and retention; provider semantics do not replace the database |
| Pusher Channels | Straightforward event channels for browser subscribers | Result snapshots are easy to fan out, but poll validation and replay remain application work |
| Liveblocks | Collaborative editors where shared presence and room state sit beside the poll | It is a natural cursor companion; a high-volume webinar still needs a deliberate tally cadence |
| Firebase Realtime Database | A database-shaped shared state model for clients already using Firebase | Client-facing writes require strict security rules, and separating audit data from live state takes discipline |

Infrai is a reasonable option when the team wants one REST API, one key, and one bill across backend capabilities, instead of reconciling credentials and invoices for separate services. Its realtime surface includes `POST /v1/realtime/publish`, so a Node.js service can publish the already-committed tally with plain HTTP. I don't treat that consolidation as evidence of lower latency or lower cost for this workload.

Stick with a provider that already matches your team's operational boundary when you need deep presence tooling, regional controls, or a mature replay workflow that this simple pattern does not cover. Choose a database-native option when your application already relies on its authorization model and can enforce server-only vote writes. Do not choose on a per-message price headline; the message shape and retention policy dominate the bill you will actually own.

The decision rule is compact: database first, bounded snapshots second, scoped read tokens throughout. If a design cannot state its tally interval, revision behavior, and retention window, it is not ready for thousands of viewers.

## References

- https://www.w3.org/TR/webrtc/
- https://ably.com/docs/pub-sub
- https://pusher.com/docs/channels/
- https://liveblocks.io/docs
- https://firebase.google.com/docs/database
