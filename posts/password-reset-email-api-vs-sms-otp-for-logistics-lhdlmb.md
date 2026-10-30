# Password Reset Email API vs SMS OTP for Logistics Sellers: 3 Evidence Checks

TL;DR: For a logistics marketplace seller, password-reset email is usually the simpler primary path because the recovery link and its delivery result can be joined to one case record. SMS OTP is a useful fallback only when the phone number is already trusted and the extra delivery, segmentation, and redaction events are worth retaining. Make the choice on evidence quality, not an assumed unit price.

The least complex design is an email reset flow with a single-use, short-lived token, a generic response to the requester, and an audit record that excludes the token. Record the request, the decision, the delivery outcome, and the completion outcome. That is enough to answer an investigation without storing the secret.

## Should a logistics seller use password reset email API or SMS OTP?

The practical question is whether the marketplace can later show what happened without reconstructing it from application logs, provider dashboards, and support notes. Email links make that chain comparatively direct: request ID, account identifier, destination hash, template revision, delivery result, link redemption, and password-change result. SMS OTP adds code issuance, attempt count, expiry, resend state, carrier-facing delivery status, and a distinction between an invalid code and a code that was never delivered.

For logistics, the evidence should prove control of the account without exposing a seller secret or full contact detail. Keep a stable recovery case ID and hash the email address or phone number with a separately managed key. The hash supports correlation; it is not a reason to retain the raw destination. Keep the policy decision that selected email or SMS, because a reviewer needs to know whether the fallback was deliberate.

The message body is not evidence. The event surrounding it is.

| Recovery path | Evidence to retain | Main operational boundary | Best fit |
| --- | --- | --- | --- |
| Password-reset email link | Request, delivery outcome, redemption, completion | Delivery acceptance does not prove inbox access | Verified seller email is the established channel |
| SMS OTP | Issuance, delivery outcome, attempts, expiry, verification, completion | Encoding can change segmentation; resend and replay rules add states | A separately verified phone closes a defined recovery gap |

I start with this table before estimating message volume. It makes the control boundary visible; a cost model that counts only sends is incomplete.

## What actually makes the bill grow?

The dominant observability term is usually retained event volume, not the reset token. Every retry, resend, callback, and verbose request log can multiply the record count. A useful planning model is:

`retained bytes = recovery cases x events per case x average event bytes x retention days`

Consider an explicitly illustrative envelope: 100,000 recovery cases, 6 events per case, and 700 bytes per structured event produce 420,000,000 bytes for one event cohort before indexing and replication. Extending retention from 30 to 90 days triples the cohort contribution if traffic is steady. The arithmetic exposes the lever: reducing noisy events or their retention changes more than arguing over a message's nominal price.

I would keep a compact, immutable decision record for the full compliance window and retain diagnostic payloads for a shorter window. The compact record contains timestamps, outcome codes, policy revision, channel, and correlation IDs. It contains no message text, reset URLs, OTP values, authorization headers, or provider response bodies. When a delivery callback matters, normalize it to a small status vocabulary and preserve the source timestamp separately from ingestion time.

That choice has a cost. A later investigation may lack original provider wording or a raw callback field. The trade-off is deliberate: raw payloads contain more personal data and create more retention surface than most account-recovery investigations require. Sampling can answer transport questions, but sampling must never remove the record needed to prove that a seller's reset completed.

## A small evidence contract beats a large log

Treat the recovery case as a state machine, even when the implementation spans an API, a queue, and a worker. Use a monotonic case ID and an idempotency key. A retry can create another delivery attempt, but it must not create another password-change authorization. On the email path, redemption consumes the token and records only a token fingerprint. On the SMS path, verification consumes the OTP and records the attempt result, never the code.

The evidence event should contain a case ID, channel, normalized outcome, destination hash, policy revision, source timestamp, and ingestion timestamp. That is a schema shape, not a claim about a particular provider. The timestamps matter because a delayed callback must not be mistaken for a late user action. The policy revision matters because recovery rules change. The destination hash matters because two events with one case ID are not sufficient if a worker sends to a different destination. Keep the collector interface internal and generic so the application can replace its storage without rewriting the recovery state machine. It should reject secrets at validation time, apply access controls by case purpose, and expose deletion or legal-hold workflows separate from ordinary application permissions.

For an implementation check, query normalized event records through the service's event-list interface and inspect the response for secrets before allowing it into a support workflow:

```bash
curl --request GET \
  --url https://api.example.invalid/v1/email/event/list \
  --header 'Authorization: Bearer ${AUDIT_TOKEN}' \
  --header 'Accept: application/json'
```

The host is illustrative; the route is the relevant interface shape. The response should contain event metadata, not reset URLs or OTP values.

## Where SMS changes the accounting

SMS OTP is operationally different even when the user experience looks short. A message can be segmented when its encoding and length require it; the character set therefore belongs in the test matrix, not in an assumption hidden inside a template. The documented distinction between GSM-7 and UCS-2 affects how character limits and segmentation are evaluated. Test the actual localized text, including punctuation and seller names, before release.

An OTP also creates a replay boundary. The service needs an expiry decision, an attempt limit, resend behavior, and a rule for whether a newly issued code invalidates an older one. Those are control decisions, and each should produce a small outcome event. Logging the full SMS is unnecessary for proving them.

Email has its own failure boundary. A reset message can be accepted by a sending system without reaching the seller's inbox, so delivery acceptance is not proof of redemption. Domain authentication and alignment are part of deliverability evidence; DKIM is specified in RFC 6376. The recovery record should distinguish accepted, bounced, expired, redeemed, and completed rather than collapsing them into “sent.”

## A decision rule for the marketplace

Use email as the default when the seller's verified email is the established account channel, the flow can record redemption, and the compliance case needs a durable explanation of the account change. Add SMS when it solves a defined gap, such as a separately verified phone channel or a policy that requires an independent factor. Do not add it because a second channel sounds safer. It brings another secret, retry path, privacy review, and event stream.

Before launch, replay these cases: unknown account, repeated requests, bounce, delayed callback, expired link, wrong OTP, resend, successful redemption, and a password change after a second browser opens the link. Check that each case has one recoverable narrative and that no raw token or code appears in application, queue, callback, or analytics logs. Then measure retention by event class.

The evidence I would stop keeping is raw message content, raw destinations, provider payloads by default, and every polling response. The evidence I would keep is the decision, normalized outcome, timestamps, policy revision, and final account-change result. That boundary makes a later compliance review possible while limiting the data exposed by a log search. **The cheaper operational design is the one that keeps the fewest records needed to explain the control.**

## Further reading

- RFC 6376, “DomainKeys Identified Mail (DKIM) Signatures”: https://datatracker.ietf.org/doc/html/rfc6376
- Twilio, “SMS Character Limits and Message Segmentation”: https://www.twilio.com/docs/glossary/what-sms-character-limit
