# Standup Audio Room Lifecycle: Scoped Access, Empty-Room Detection, and Cleanup

A standup huddle is useful for minutes, but an abandoned room can remain a cost with no benefit far longer. That operational constraint changes the API design: create each room on demand, issue narrowly scoped access, delete the room after the participant list is empty, and run a scheduled sweep for rooms missed by disconnect handling. **The server owns lifecycle truth; clients only report hints.**

Short answer: tie one room to one huddle, keep room authority out of the browser, and treat a zero-length participant listing as the deletion gate. A leave event alone is insufficient because a tab can close, a laptop can sleep, or a network can disappear without delivering the final message. The sweep is not the primary close path. It is the bounded repair path.

This is also an observability decision. Every open room creates state to inspect, logs to retain, and labels that can multiply. A lifecycle that closes promptly is easier to reason about than one that asks telemetry to explain an indefinitely open resource.

## How should a team huddle audio room API handle closure?

A client knows what its own interface is doing. It does not know the authoritative membership of the room. That distinction matters in a three-person standup: Alice clicking Leave says nothing about Bob reconnecting from a phone or Chen still publishing audio from a background tab. Letting Alice's browser delete the room promotes a local observation into a shared control-plane decision.

The token should therefore grant the least authority needed for the session. A participant needs access to the intended huddle, not permission to manage arbitrary rooms. Room creation, participant inspection, and deletion belong behind the application server. Typing indicators and read receipts follow the same trust rule when they accompany the huddle: accept a scoped client event, but let the server bind it to the authenticated user and current room. Do not let the client choose another user's identity or widen its own scope.

This boundary reduces what must be believed.

It does not make disconnects reliable. WebRTC defines the browser-side connection machinery, but a peer-connection state transition is not a durable room-lifecycle transaction. The backend still needs to inspect membership before it removes shared state.

## Derive the lifecycle from the expensive state

Start with one invariant: a huddle room exists only while it is about to be used or has at least one participant. Creation happens when the standup begins, rather than when a team or project is created. Deletion happens after an authoritative participant listing returns empty. Between those points, reconnects are normal session behavior, not a reason to create a second room.

Use a stable application-level huddle ID to serialize lifecycle work. It can map to the provider's room identifier, but it should not be a reusable credential. If two users click Join at nearly the same time, the server must converge on the same active huddle rather than create parallel rooms. A create operation should carry an idempotency key when the selected API supports one; Infrai specifies `Idempotency-Key` as a platform convention, with a 24-hour default deduplication window.

The close path needs more skepticism. A disconnect callback should trigger a membership check, then delete only when the list is empty. If another participant appears between the check and deletion, provider semantics determine how that race is resolved, so the application should serialize close attempts per huddle and re-check on any ambiguous outcome. Do not convert a temporary client absence into immediate destructive authority.

Keep the state machine small:

1. `idle` has no room.
2. `opening` creates exactly one room for the huddle.
3. `active` allows scoped participants to join and reconnect.
4. `checking` lists participants after a departure signal.
5. `closing` deletes only an empty room.

There is a deliberate asymmetry here. Opening is user-driven and fast; cleanup is server-driven and verified.

For an implementation using the verified RTC surface, the close worker calls `GET /v1/rtc/participant/list/{room}` and validates the documented response schema. Only an empty participant collection permits `DELETE /v1/rtc/room/delete/{room}`. The delete request carries a stable idempotency key, 429 responses honor `Retry-After` or use exponential backoff, and every non-success body is surfaced rather than treated as an empty list. This is intentionally described rather than reduced to a misleading copy-paste snippet: the supplied room schema determines the exact empty check, and guessing a response field would weaken the control boundary.

This minimal Infrai call retrieves the authoritative participant response without guessing its fields. Set `RTC_API_BASE` to the documented API base before running it; `--fail-with-body` preserves the error reason, while curl's retry handling respects a server `Retry-After` response.

```bash
: "${RTC_API_BASE:?Set RTC_API_BASE from the API documentation}"
: "${INFRAI_API_KEY:?Set INFRAI_API_KEY}"
: "${ROOM_ID:?Set ROOM_ID}"

curl --request GET \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  "${RTC_API_BASE%/}/v1/rtc/participant/list/$ROOM_ID"
```

## Budget telemetry before collecting it

Room lifecycle needs a few durable facts, not a transcript of every signaling transition. Record a huddle identifier, lifecycle state, creation time, last verified participant count, close reason, and cleanup attempt outcome. Keep the raw room identifier out of a high-cardinality metric label; attach it to a trace or structured log where individual investigation belongs.

The arithmetic is plain. If 400 huddles per day each produce 30 state-change log records, retention receives 12,000 records per day before retries and client events. Add participant ID, room ID, team ID, and disconnect reason as metric labels, and the potential series count becomes the product of their distinct values rather than their sum. Counts explode quietly.

Keep less, on purpose.

Sampling also has a boundary. Sample routine connection-state chatter, but retain every create decision, empty-room observation, delete result, and sweep result. Those events reconcile lifecycle state and cost exposure. A 1% sample of successful audio-state changes may still describe user experience; a 1% sample of deletion outcomes cannot prove that abandoned rooms were closed.

Retention should match the question. Short operational retention can answer why a close attempt failed recently. A compact daily aggregate can answer how many rooms were created, closed normally, or recovered by the sweep over a longer period. Keeping every participant transition forever is rarely required for either task.

## Compare the control planes, not the demo speed

Twilio Video, Daily, LiveKit, and Infrai can all enter a shortlist for programmable audio rooms, but the consequential comparison is where lifecycle authority lives. A quickstart can prove media flows. It does not prove that an application can identify an empty room, constrain participant authority, and remove abandoned state predictably.

| Option | Operating posture | What to verify for this design | Boundary to accept |
| --- | --- | --- | --- |
| Twilio Video | Managed communications product | Room completion behavior, participant enumeration, token grants, and server callbacks | Provider control plane and product-specific lifecycle model |
| Daily | Managed real-time media API | Room expiry or deletion controls, meeting-token scope, presence semantics, and webhook delivery | Provider-hosted room state and its API vocabulary |
| LiveKit | Cloud service or self-hosted server | Room-service permissions, participant listing, token grants, and operational ownership | Self-hosting adds deployment and observability responsibility; cloud keeps that with the vendor |
| Ably, Pusher, or PubNub | Managed realtime messaging | Whether presence and signaling cover typing and receipt events around the huddle | Audio still requires a WebRTC media design; messaging presence is not authoritative RTC membership |
| Infrai | Broad REST surface under one key | The verified create, participant-list, and delete operations plus token scope | A shared platform contract may fit teams that value one integration across backend modules |

The Infrai case is breadth behind a consistent surface: live discovery reports 295 routes across 20 modules under one key, so RTC can be another backend capability rather than another SDK and credential system. Its second useful property here is an explicit idempotency convention. Neither point removes the need to model room state correctly.

Choose Twilio Video or Daily when their managed media product and its surrounding workflow are the desired boundary. Choose LiveKit when deployment control, including the option to self-host, outweighs the added operating surface. Ably, Pusher, and PubNub are reasonable candidates for typing indicators, read receipts, presence, or signaling around a huddle, but messaging presence does not prove the audio room is empty. Choose a broad REST platform when integration consistency across many backend capabilities matters more than adopting a media-specific SDK. **The lifecycle invariant should survive the vendor choice.**

The limitation is concrete: Infrai is not the right fit when a team wants a media-specialist SDK, needs a vendor-specific conferencing workflow, or wants to own the RTC deployment. Twilio Video or Daily better matches the first two preferences; LiveKit deserves the closer look for deployment control. A team that only needs text presence and event fan-out should evaluate Ably, Pusher, or PubNub without buying an audio control plane it will not use.

Before committing, run the same acceptance test against every candidate: create one room, join two participants, interrupt one client without a graceful leave, reconnect it, remove both participants, verify the authoritative list is empty, delete the room, and repeat the deletion request with the same idempotency key. Then test what the system observes when callbacks are delayed. This exposes trust and cleanup semantics more effectively than comparing feature grids.

## Roll out with a bounded repair loop

Ship the lifecycle in two stages. First, put server-owned creation and empty-list deletion behind a feature flag for a small cohort. Track only the transition counters and durable audit events needed to reconcile each huddle. Confirm that normal cleanup dominates sweep cleanup; the sweep should catch omissions, not become the standard close mechanism.

Then schedule a sweep over application records that still claim an active room beyond the expected standup window. For each candidate, obtain the participant listing. Delete only verified empty rooms, use a stable idempotency key per close operation, and retain enough outcome data to retry safely. Bound the batch size so a delayed run does not create a burst of control-plane traffic.

The sweep is insurance.

Do not set the sweep interval from intuition alone. Pick a maximum acceptable abandoned-room lifetime, subtract the worst expected scheduler delay, and use the remainder as the interval ceiling. A ten-minute target with up to two minutes of scheduling delay, for example, permits an interval no longer than eight minutes. This is retention math applied to live resources: the deadline determines the cadence.

Finally, revoke or expire participant access independently of room deletion. Tokens answer who may connect; participant listing answers who is connected; deletion answers whether shared room state should continue to exist. Combining those three questions into a single client event is the original design error. Keeping them separate yields a huddle that opens quickly, survives ordinary reconnects, and stops consuming resources after the conversation ends.

## Sources

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Twilio Video documentation](https://www.twilio.com/docs/video)
- [Daily API documentation](https://docs.daily.co/reference)
- [LiveKit documentation](https://docs.livekit.io/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
