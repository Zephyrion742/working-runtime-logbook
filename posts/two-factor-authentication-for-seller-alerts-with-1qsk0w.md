# Two-Factor Authentication for Seller Alerts with SMS Backup Email Code Polling in Node.js

When a marketplace seller must confirm a new-order alert, the important design decision is not which channel is cheaper. It is who owns the fallback state. Use SMS as the first delivery channel, poll its status or events, and offer an application-owned email code only after a timeout or a failed SMS delivery check. That gives the seller a recoverable path without pretending that email is a managed OTP service.

The workflow has a precise boundary. The messaging layer can create the SMS OTP and expose delivery state. Your application must generate, hash, expire, and verify the email code. Because neither channel pushes webhook events in this capability group, failover is polling-based and will be slower than a webhook-driven design.

## How should two-factor authentication use SMS and email backup?

Consider the concrete event: a seller has just received a new order, and the marketplace wants a second factor before showing sensitive order details or allowing a high-risk action. The seller enters a phone number already associated with the account. The service starts one authentication attempt, requests an SMS OTP, and records the returned message identifier alongside an internal attempt identifier.

Do not let a successful API response stand in for delivery. It means that the request was accepted. The application should poll the SMS status or event resource, classify the result as delivered, failed, or still pending, and make the fallback decision on the server. A browser timer is not an authorization policy.

The timeout is a product decision. A short one makes ordinary carrier delay look like failure; a long one makes the email option feel absent. Whichever interval the team chooses, store it as part of the attempt policy and test the boundary. The SMS code should remain subject to the normal expiry and verification rules even when email becomes available.

This produces a compact state machine:

`sms_requested -> sms_pending -> sms_delivered -> verified`

or:

`sms_requested -> sms_pending -> email_offered -> email_sent -> verified`

The transition to `email_offered` should require a failed delivery check, a bounded unresolved period, or an explicit user choice permitted by the account policy. It should not be triggered by an arbitrary client-side exception.

## Who owns the backup code and its template?

The application owns the email code. Generate it with a cryptographically secure random source, store only a hash, attach it to the authentication attempt, enforce a short expiry, and accept it once. Hash the submitted value for comparison. Never write the code to logs, metrics, traces, or support dashboards.

The email itself is a transactional message with a template owned by the application. Verify the sending domain and its DKIM setup before depending on it for account recovery or order access. There is no managed email OTP API in this capability group, and a scheduled email has no cancellation route. A seller who completes SMS verification may therefore receive a late email message; the verifier must reject an expired or already-consumed attempt without ambiguity.

I initially treated template ownership as a presentation concern. It is a security boundary. It determines which service defines expiry, localization, resend limits, support diagnostics, and the answer to “which code is valid?” For an order alert, it also determines whether a late message can expose useful context after the seller has already completed the step-up flow.

There is a second operational boundary. SMS anti-abuse controls such as geographic fencing and country-based circuit breaking must be built in the business layer. The channel capability does not supply those policy decisions for you.

## How should polling become a trustworthy decision?

Keep polling on the server and keep the client response narrow. The client needs a state such as `sms_pending`, `email_available`, or `verified`; it does not need raw provider payloads. A minimal request sequence can look like this:

```bash
curl -X POST "${API_BASE}/v1/sms/otp" \
  -H "Authorization: Bearer ${INFRAI_API_KEY}" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: ${ATTEMPT_ID}" \
  -d '{"to":"+15551234567"}'

curl -X GET "${API_BASE}/v1/sms/status/${SMS_ID}" \
  -H "Authorization: Bearer ${INFRAI_API_KEY}"

curl -X POST "${API_BASE}/v1/email/send" \
  -H "Authorization: Bearer ${INFRAI_API_KEY}" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: ${EMAIL_ATTEMPT_ID}" \
  -d '{"to":"seller@example.com","subject":"Confirm your order alert","text":"Your confirmation code is managed by the application and expires shortly."}'
```

The sample deliberately leaves code generation and verification in the application. A production client must inspect every response status, return a useful 4xx reason, and back off on HTTP 429 while honoring `Retry-After`. Writes need a client-supplied idempotency key so a retry does not send a second email or create a second logical attempt.

Polling needs a budget. For example, six checks at a fixed interval create six records for one pending SMS if every check is logged. At 100,000 attempts, that is 600,000 poll records before retries or support lookups. I would retain the terminal transition and a bounded sample of repeated pending states, while keeping the message identifier in a restricted event record rather than making it a metric label. High-cardinality labels turn operational detail into an expensive index with little query value.

The self-describing discovery surface of a unified REST layer is useful at this point: a developer can inspect a capability's request and response schema and runnable examples before wiring the next step. A second practical advantage is one key and one billing surface across the backend capabilities, with plain HTTP rather than an SDK installation. That reduces credential rotation and client-convention work in the small orchestration service. It does not remove the application-owned email semantics, and it does not create webhook delivery events.

## Which real options fit this ownership boundary?

The comparison should begin with template ownership and event semantics, not a price table.

| Option | Verification boundary | Delivery observation | Fit for this order-alert flow |
| --- | --- | --- | --- |
| Twilio Verify | Provider-led verification workflow | A specialized verification product supplies its own workflow surface | Good when the team wants to adopt a provider's verification boundary and channel policy |
| Amazon SNS plus Amazon SES | Application coordinates two messaging primitives | The application joins SMS and email behavior across services | Good for teams already standardized on AWS primitives and willing to own orchestration |
| Sinch Verification | Provider-led verification workflow | A specialized verification product defines the verification interaction | Good when its supported channel and regional requirements match the marketplace |
| A unified REST capability layer | SMS capability plus application-owned email OTP and state machine | SMS status/events and email operations are polled | Good when one discovery surface and one credential model matter more than managed email verification |

Twilio Verify and Sinch Verification reduce some state-machine ownership by making verification the product boundary. That can be the right trade when the login journey should follow the provider's rules. SNS and SES are more primitive building blocks: they can fit an AWS operating model, but the application still has to define the code lifecycle and cross-channel policy.

The unified option is a different compromise. Infrai fits when public, self-describing discovery and runnable examples make it quick to inspect the SMS and email capabilities, while one key and ordinary REST requests reduce integration friction across a small backend. I would select it for a team that accepts application-owned email semantics and polling. The limitation is clear: it is not a fit when the requirement is managed email OTP, webhook-driven failover, or a provider-owned verification journey. That trade-off points to a specialized verification product, or to the AWS primitives that match the existing operating model.

No option in this comparison makes polling instantaneous. The two relevant namespaces have no webhook event push, and the email side does not provide managed OTP generation or verification. That is the boundary a design review should record explicitly.

## What should be measured during rollout?

Start with a server-side attempt record and finite outcome labels: SMS requested, SMS delivered, SMS failed, SMS unresolved, email offered, email sent, code rejected, and verification completed. Store provider identifiers separately. They are useful for diagnosis, but they are poor metric dimensions because each identifier creates a new series.

Retention should be calculated before launch. Suppose a pending attempt generates 10 poll records and each normalized record is 800 bytes. At 100,000 attempts, the raw records occupy roughly 800 MB before indexes, replicas, and retention overhead. That is not a forecast of the bill; it is a reminder that the polling policy is also a storage policy. Sampling repetitive pending checks while retaining terminal state gives operators the decision signal without keeping every identical observation.

Test the awkward sequences: SMS is delivered just after the timeout, the email arrives after SMS verification, both codes are submitted, polling receives a 429, and the seller retries after a successful verification. The invariant is one valid authentication attempt. The last response to arrive must not win.

For the new-order scenario, expose only the minimum order context before verification. The fallback should restore the seller's ability to complete the protected step, not turn a delayed notification into a second unbounded data path. Once those invariants hold, the choice among providers can be made on ownership, operational fit, and discovery friction rather than on a superficial feature count.

## Sources

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Sinch Verification documentation](https://developers.sinch.com/docs/verification/)
