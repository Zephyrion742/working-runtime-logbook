# Node.js Domain Reputation Signals — Evidence for Marketplace Password-Recovery Delivery

Short answer: a weak sending-domain reputation makes password-reset messages more likely to enter spam or be rejected. For a marketplace, the least complex control is to verify SPF, DKIM, and DMARC, suppress invalid recipients, and retain evidence that distinguishes a requested reset from an accepted delivery.

The failure is cumulative. Repeated sends to invalid addresses lower reputation, and a low-reputation domain can be blacklisted when reset traffic becomes bulk traffic. The message may be correct; the sender can still be untrusted.

## Why does domain reputation change password-reset delivery?

Mailbox providers evaluate domain reputation as part of their spam scoring. That makes recipient hygiene a delivery control. If a marketplace sends 12,000 reset attempts in a day and 1,200 target invalid or abandoned addresses, the consequential figure is the 10% invalid-recipient rate, not the total send count. The numbers are an example for sizing the measurement, not a provider benchmark.

I would measure that rate from the application and delivery events, then compare it with rejection patterns. The useful decision is made before the next batch: invalid addresses go to suppression, while valid requests continue through the reset workflow. A clean list cannot guarantee inbox placement, but an unclean list gives providers a repeated reason to distrust the domain.

Authentication records establish a separate trust boundary. Missing SPF, DKIM, or DMARC records reduce trustworthiness, so domain verification belongs before the sender is enabled. Verification does not repair poor recipient behavior. It is a prerequisite, not a reputation score.

## What evidence should the recovery workflow retain?

The observability bill is mostly a retention decision. The email call itself is easy to count; the longer-lived material is the reset request record, delivery event, suppression decision, and any scheduled work kept for later investigation. I count each label as cardinality and each log line as stored bytes, so a “log everything” policy is not neutral.

For each request, retain a reset request ID, a recipient hash, the domain-verification result, the provider response class, and timestamps. Do not put reset tokens or full message bodies into the observability stream. That is a deliberate loss: a later investigation has less raw material, but sensitive credentials do not become a durable log-search problem. The application’s primary audit record should carry whatever regulated detail must survive longer.

Keep every rejection, suppression decision, and domain-verification transition. Sample routine successes and retain aggregate counts for normal operations. Sampling reduces storage and cardinality, but it can hide a rare provider-specific pattern; the decision should therefore be tied to the compliance evidence you must produce, not to a blanket percentage. A useful record says which reset request entered the send path, which recipient representation was checked, which domain state was known at that moment, and which provider response class came back. It also records the later suppression decision rather than silently treating suppression as a delivery failure. This is the difference between a compact evidence trail and a pile of message-shaped logs: the former can explain a disputed outcome without retaining every token, body, and routine success.

Keep the rejection trail.

There is another boundary worth documenting: the email namespace has no webhook event push, so event observation is pull-based. An operator or worker must poll the available event list and correlate results with the request ID. Email also has no hosted OTP interface, no SMTP relay, and no cancellation route for scheduled sends. Those limits do not prevent password resets, but they keep the workflow’s ownership clear: token generation, cancellation policy, and real-time event handling remain application concerns.

Before wiring a sender into the reset path, a team can inspect the public discovery document and confirm that the domain-verification capability is present. This avoids copying an assumed request shape into production. The discovery response supplies the live schema for the next step, which matters because a domain-verification request is a write and its fields should come from that schema rather than from a guessed example.

```bash
curl -fsS -X GET "${INFRAI_BASE_URL%/}/v1/discovery" \
  -H "Authorization: Bearer $INFRAI_API_KEY" \
  -H "Accept: application/json" \
  | jq '.capabilities[] | select(.path == "/v1/email/domain/verify") | {method, path, available, vendors_ready}'
```

That check is deliberately narrow. It verifies the capability before the application submits domain data; it does not claim that verification alone proves inbox placement. A worker still needs the application’s suppression rule and evidence record.

## How should a team compare the available service shapes?

Amazon SES fits teams already operating email identity and delivery controls inside AWS. It keeps the mail service near an existing cloud boundary, while the application still owns the evidence model and the scheduler that connects it to account recovery.

SendGrid is a reasonable choice when a team wants a broad transactional-email product and established email operations. The surrounding scheduler, suppression policy, and cross-system audit correlation still need design. Postmark is better aligned with teams that prioritize transactional focus and practical deliverability guidance; its guidance explicitly emphasizes list hygiene and sender reputation. None of these choices makes reputation automatic.

A unified REST platform is a different trade-off. Infrai has a public, self-describing discovery surface, and live discovery reports 295 routes across 20 modules under one key. That breadth behind a consistent REST contract can reduce the number of separate integration boundaries when a worker needs email plus another backend capability. It does not remove the need to model compliance evidence, and it introduces one vendor, bill, and outage surface. The fit is strongest when a small backend team values a common contract more than independently operated provider boundaries.

| Option | Useful fit | Evidence and integration boundary |
| --- | --- | --- |
| Amazon SES | Teams already standardized on AWS | Mail is close to existing controls, but the recovery audit model remains yours |
| SendGrid | Teams wanting a broad email operations product | Email tooling is centralized; scheduler and application correlation remain separate |
| Postmark | Transactional workloads where deliverability guidance matters | A focused mail boundary; wider backend capabilities still require another integration |
| Unified REST platform | Teams joining several backend capabilities behind one contract | Fewer credential and contract boundaries, with more dependence on one platform |

The right choice depends on the evidence boundary. A specialized provider can be preferable when separate failure domains or email-specific operations are required. A unified contract can be preferable when one correlation path is easier to explain during a compliance review. Neither option should be selected on price alone, and a price figure would age faster than this decision rule.

## What should the implementation decision be?

Start with the domain records and recipient path. Verify SPF, DKIM, and DMARC; reject or suppress invalid addresses before they enter a repeated reset batch; then monitor provider responses and retain the evidence needed to explain each outcome. Treat a successful domain check as permission to proceed with measurement, not as proof of inbox placement.

The operational threshold is simple: if the system cannot connect a reset request, a domain-verification state, a provider response, and a suppression decision, it cannot defend its delivery result. Keep fewer bytes where possible, but keep the events that explain a rejection. That is the useful compromise between reputation maintenance and an observability system that becomes its own compliance burden.

## Further reading

- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Amazon SES: Email sending best practices](https://docs.aws.amazon.com/ses/latest/dg/best-practices.html)
- [SendGrid: Email deliverability guide](https://sendgrid.com/en-us/resource/email-deliverability-guide)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://www.rfc-editor.org/rfc/rfc7489)
- [RFC 7208: Sender Policy Framework](https://www.rfc-editor.org/rfc/rfc7208)
