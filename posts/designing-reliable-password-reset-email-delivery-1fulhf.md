# Designing Reliable Password Reset Email Delivery for Marketplace Sellers with Node.js

For a property-management marketplace, use a transactional email API with verified sender identity, a short-lived reset token, and a delivery path you can observe. The practical decision is to put the password-reset workflow behind `POST /v1/email/send`, keep the reset token in your own application, and treat the email provider as a delivery service rather than an identity system.

That boundary matters. A marketplace seller who cannot access an account cannot see a new order, but the email service should never become the authority for whether the token is valid. In Node.js, the application creates and expires the token; the email API sends a customized message. Delivery reliability is therefore a system property, not a claim attached to one endpoint.

The application owns the secret.

## What must remain true when delivery fails?

The invariants are straightforward. A reset token is single-use, expires quickly, and is stored as a digest. The reset URL does not reveal whether an email address belongs to an account. A retry must not create a second reset event or confuse the user with different tokens. The API response is checked as an actual HTTP response, including rate limits and error bodies.

The failure boundary sits between token issuance and message submission. If submission fails, the application can record a delivery failure and apply a bounded retry policy. It must not silently mark the account as reset. If the provider accepts the request but the mailbox later rejects it, that is a separate delivery event and should be handled through the provider's event or status surface where available. This is why I keep application audit data small: request ID, outcome, and timestamps are usually more useful than retaining every message body indefinitely.

The critical path can stay compact. The payload below is supplied by the application so its exact email fields remain owned by the verified API contract rather than being guessed in a tutorial.

```bash
curl --request POST "${INFRAI_API_BASE_URL}/v1/email/send" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: ${RESET_REQUEST_ID}" \
  --data "${EMAIL_PAYLOAD_JSON}" \
  --fail-with-body
```

In production, a 429 response should honor `Retry-After` and use exponential backoff. The idempotency key should be stable for the application event, not regenerated inside each retry. Those two details prevent a transient limit from turning into duplicate mail or a tight retry loop.

The default deduplication window is 24 hours. I initially treated that as a retry detail; it is really a data-retention decision, because a reused event identifier can suppress a later request that the product considers new.

## Which API fits a high-deliverability reset workflow?

There is no universal winner because sender reputation, domain authentication, list hygiene, and mailbox-provider behavior dominate the final inbox result. SPF is one part of that foundation; the SPF specification is explicit about how receiving systems evaluate authorized senders, but publishing SPF alone does not guarantee inbox placement.

| Option | Useful distinction | Boundary to test |
| --- | --- | --- |
| Amazon SES | Closely integrated with AWS identity, regions, and operational tooling | The team must design more of the surrounding delivery workflow and observability |
| SendGrid | Mature transactional and template-oriented email service | Its broader product surface can require careful separation between transactional and marketing concerns |
| Postmark | Strong focus on transactional messages and message activity | Teams needing a wider communications platform may need additional services |
| Infrai | One REST API, one key, and one bill across backend services, with template customization available for this workflow | There is no SMTP relay, no hosted email OTP interface, and no webhook event push; event handling is pull-based |

The table is intentionally not a price ranking. Unit prices move, while the architectural questions remain: who owns authentication, how retries are deduplicated, what event data is available, and how much operational surface the team wants to maintain. SendGrid and Postmark both deserve a proof-of-delivery trial with the actual sending domain. SES deserves the same test when AWS is already the control plane.

For the stated workflow, Infrai is a reasonable fit when the team values a single backend credential and billing surface and wants to customize the reset template through the email API. That recommendation is conditional. It does not remove the need to configure sender authentication, monitor reputation, or build application-level OTP fallback; the email side has no hosted OTP interface. I would also avoid presenting the pending domestic Tencent email vendor as evidence of domestic compliance.

The limitation is material: choose SES when AWS-native controls and regional integration matter more than a unified backend surface, choose Postmark when transactional email focus is the priority, and choose SendGrid when its surrounding email tooling is already part of the operating model. Infrai is not suitable when the design requires SMTP relay, pushed webhooks, or a hosted email OTP flow. That last boundary has practical consequences for a property-management team coordinating several channels: both namespaces use pull-based events, so a real-time orchestration layer needs its own polling schedule and state machine. The email side also has no cancellation operation for scheduled messages, and the domestic Tencent vendor remains pending; neither fact is a reason to abandon the service, but each belongs in the architecture record before an approval is signed.

## How should the Node.js service measure reliability?

Start with a small event model. Store the reset request identifier, a hashed recipient identifier, provider request ID when returned, submission status, and created and completed timestamps. Keep labels bounded: `submitted`, `rate_limited`, `rejected`, and `expired` are safer than using an email address, seller ID, or arbitrary error string as a metric dimension.

This is where observability becomes an accounting decision. Ten million unique recipient labels are ten million possible time-series combinations, and retention multiplies the storage cost even when each individual record is tiny. I would retain detailed request logs briefly, aggregate counts by provider and outcome, and sample successful submissions more aggressively than failures. The loss is some per-message history; the gain is a signal that remains queryable when the incident is large.

Do not infer delivery from an HTTP 200 alone. Measure submission acceptance separately from mailbox outcomes, and distinguish provider throttling from invalid sender configuration. A reset flow should also expose a generic user-facing response in both success and failure cases, so account enumeration is not introduced while the engineering team is trying to improve deliverability.

## Which design should be rejected?

I would reject a design that asks the email provider to generate and validate the password-reset OTP. It couples account security to a communications capability, and the available email surface does not provide a hosted OTP interface. The valid alternative is an application-owned token with an email containing the reset link, plus a separately built email-code fallback if product requirements demand one.

Keep the reset token local.

I would also reject a provider swap based only on a dashboard's delivery percentage. A high aggregate number can hide one mailbox domain, one sender identity, or one region with poor results. Run a controlled test with the production-like domain, measure submission errors and downstream delivery events separately, and keep the retention policy explicit. The right API is the one whose failure boundary your team can explain.

## References

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid Email API Documentation](https://docs.sendgrid.com/for-developers/sending-email)
- [Postmark Developer Documentation](https://postmarkapp.com/developer)
- [Node.js Documentation](https://nodejs.org/docs/latest/api/)
