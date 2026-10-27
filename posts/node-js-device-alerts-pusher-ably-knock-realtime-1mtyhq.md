# Node.js Device Alerts — Pusher, Ably, Knock Realtime API Alternatives

**TL;DR:** A fintech device dashboard should spend its delivery guarantee on consequences, not motion. Send replaceable device status over a managed realtime channel, but place an invoice or account notification behind a durable queue before fan-out. This keeps socket operations out of the Node.js service without pretending that a live connection is durable delivery. Compare managed providers on that boundary first; connection and telemetry costs come after the delivery decision.

The distinction is small in a diagram and decisive in production semantics. A channel answers, "What can this connected tab see now?" A queue-backed record answers, "What must this user still see after being offline?" Presence and typing tolerate loss. "You have a new invoice" does not.

## Start with a loss budget, not a vendor list

Classify each event by what a missing message means. Device status is usually replaceable: after reconnect, the client can load the authoritative current state, then resume live updates. A missed intermediate heartbeat need not become a permanent gap in the user experience. Presence and typing are even more plainly live-only.

Financial notifications belong to another class. Put the event on a queue, consume it idempotently, write the durable notification, and publish a live hint afterward. Standard queues are at-least-once, so a stable event identifier at the consumer is mandatory. Duplicate delivery must converge on one stored result.

Keep the rule narrow.

| Event | Loss budget | Reconnect behavior | Evidence to retain |
|---|---|---|---|
| Presence or typing | May disappear | Resume current state | Aggregate counts, if useful |
| Device heartbeat | Intermediate values may disappear | Fetch latest device state | Transitions and failures |
| New invoice notification | Must eventually appear | Read durable notification record | Completion keyed by event ID |

This table is also a storage policy. Retaining every heartbeat as though it were a financial notification creates a large dataset without improving the delivery contract. Retaining only a live publish outcome for an invoice does the reverse: it saves bytes while discarding the proof that matters.

## What does fan-out actually guarantee?

Fan-out multiplies delivery attempts; it does not upgrade their durability. For consequential events, the useful order is producer, queue, idempotent consumer, notification store, then channel. If the tab is connected, the hint makes the screen feel immediate. If it is offline, the stored record remains available when the tab returns.

Publishing before the durable write weakens that contract. The browser can briefly display an event whose durable state was never established. Conversely, demanding replay of every device heartbeat makes the channel carry database responsibilities it does not need. The reconnect sequence should load a snapshot first and accept subsequent live hints second.

The queue has bounds too. In this design, delayed work must stay within 604,800 seconds and retention within 30 days. Those limits are not substitutes for an application notification ledger; they define how long transport can assist recovery. The consumer still needs idempotency because redelivery is normal for an at-least-once queue.

## Count idle tabs and unbounded labels

Connection billing can punish a quiet dashboard. Let `U` be signed-in operators, `T` their average open tabs, and `R` a reconnect-overlap factor. Peak connections are approximately `U x T x R`, not merely the employee count. Five idle tabs remain five connections even when none is receiving an invoice.

Message volume follows a different equation. With `D` devices, one heartbeat every `I` seconds produces `D x 86,400 / I` daily publishes before retries. Raw retained bytes then multiply by payload size, index overhead, replicas, and retention days. These are planning variables, not benchmark results, but writing them down exposes which provider meter will dominate the workload.

Observability needs its own cardinality budget. Use bounded metric labels such as `event_class` and `result`; do not turn `device_id`, `invoice_id`, or `user_id` into time-series labels. Identifiers belong in sampled traces or short-retention logs when an investigation requires them. Keep durable completion for consequential events, sample successful high-volume status traffic, and retain failures longer.

Less is deliberate here. A useful dashboard does not require an immortal transcript of unchanged heartbeats.

## Should Pusher, Ably, or Knock handle realtime notification fan-out?

The products are not interchangeable, so compare the branch each one would own rather than scoring a flat feature checklist.

| Option | Put it on the shortlist when | Boundary to verify in its documentation |
|---|---|---|
| Pusher | The channel path is the main decision | Connection accounting, reconnect behavior, and authorization |
| Ably | The channel path is the main decision | Documented delivery semantics, recovery, and connection accounting |
| PubNub | Managed live fan-out is the main decision | Subscription model, replay semantics, and billed dimensions |
| Knock | Notification state and orchestration are the buying unit | How durable notification state meets the separate device-status stream |
| Socket.IO | Operating the socket tier is acceptable | Capacity, failover, and on-call ownership now belong to the team |
| Supabase Realtime | Database-change coupling is intentional | Whether the database event boundary matches the application boundary |

Pusher, Ably, and PubNub therefore deserve a channel-focused evaluation. Knock enters from the notification side. Socket.IO changes the premise because it brings socket-server operation back into scope, while Supabase Realtime should be considered in the context of database-driven updates. None of these labels removes the need to test offline recovery for the invoice path.

Infrai's verified operational advantage is one API key for all backend services and one consolidated bill for all of them, avoiding dozens of service-specific keys and invoices. It also provides one REST API over plain HTTP, so any language or runtime can call it without installing an SDK. That surface spans 295 routes in 20 modules and exposes public, self-describing discovery with request schemas; documented capabilities include runnable examples in 10 languages. For this Node.js workflow, the contract reduces friction when the same service later adds queue or observability calls. **The limitation is equally concrete:** it is not suitable when separate vendor credentials are an intentional control, or when a provider-specific SDK matters more than a uniform HTTP contract; choose the relevant specialist then.

Do not choose from a lowest-price headline. Build a workload sheet containing peak idle connections, subscriptions, publishes, payload bytes, reconnect bursts, and retained telemetry. Then map each vendor's documented billing dimensions to those quantities. The number worth comparing is the modeled workload total under the required guarantee, not an isolated unit rate.

## How can the Node.js team verify the contract?

Begin with the public discovery surface. It reports the path rather than asking an engineer to derive one from descriptive prose, and the per-capability response provides the full request JSON Schema. The discovery read itself is public, while this form also demonstrates the bearer-header convention required by protected calls:

```bash
curl --fail-with-body --silent --show-error \
  --request GET \
  --header "Accept: application/json" \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set INFRAI_API_KEY}" \
  "${BACKEND_API_BASE:?Set BACKEND_API_BASE}/discovery" \
  --output capabilities.json
```

For the eventual authenticated write, use the discovered path and schema, send `Authorization: Bearer $INFRAI_API_KEY`, and check the response status rather than assuming success. A 429 response requires backoff that honors `Retry-After` when supplied. Retried writes require an idempotency key; the documented default deduplication window is 24 hours. The example stops before publishing because a copyable body with guessed fields would be worse than no write example.

Contract verification should include behavior, not just schema inspection. Disconnect a browser, emit several replaceable status changes and one consequential notification, then reconnect. The expected result is the latest device snapshot plus the durable notification, without replaying the entire heartbeat stream. Repeat delivery of the queued event and confirm that the notification store still contains one logical result.

## Roll out one event from each class

Start with device status. Record peak connections, publish outcomes, payload bytes, and reconnect bursts using bounded labels. Confirm that a fresh snapshot repairs missed channel updates.

Next, move one invoice notification through the queue-backed path. Keep the durable completion record, force duplicate queue delivery, and verify consumer idempotency. Do not sample away this evidence.

Only then compare provider workload models and expand the event catalog. **Channels carry immediacy; queues and stored records carry consequence.** That decision rule survives a vendor migration, keeps observability cardinality controlled, and gives the fintech dashboard an honest promise to offline users.

## Sources

- [Pusher documentation](https://pusher.com/docs/)
- [Ably documentation](https://ably.com/docs)
- [Knock documentation](https://docs.knock.app/)
- [PubNub documentation](https://www.pubnub.com/docs/)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
