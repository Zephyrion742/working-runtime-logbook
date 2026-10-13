# Healthtech Welcome Flow Email API — Custom Domain, DKIM, Suppression, Polling

The least complex acceptable design is a transactional email API behind a narrow application-owned contract: check suppression, send the generated report attachment from an authenticated domain, retain one delivery record, and poll status on a schedule. **TL;DR:** choose this pattern when US/EU SaaS onboarding can tolerate delayed delivery evidence. The stable contract matters more than the provider SDK because it lets the implementation behind that capability move without changing report-generation code.

Start with the bill. For this workflow, the dominant controllable observability term is the number of event records retained, multiplied by their stored bytes and retention time. Vendor price should not lead the decision. A report service that preserves every poll response can turn one email into many nearly identical records; preserving one normalized state transition per change keeps the evidence useful without making polling frequency equal storage growth.

## What should the report service retain?

Use a planning model before choosing a vendor. Suppose a system sends 10,000 reports per day, polls each message 12 times, and stores a 2 KB response for 30 days. That policy retains 3.6 million poll documents, or about 7.2 GB before indexes and replicas. These are illustrative inputs, not vendor measurements. If the service instead stores one 300-byte normalized record for each of three assumed state changes, the same workload retains 900,000 records, about 270 MB before indexes and replicas. The useful change is deduplication by state, not a cheaper request.

Count labels too. `tenant_id`, `message_id`, recipient address, and report identifier are high-cardinality values; putting them into metric labels creates a series for nearly every delivery. Keep them in a bounded audit record or trace attribute, and expose low-cardinality metrics such as region, terminal state, provider class, and attempt bucket. Retain the report generation ID, provider message ID, suppression decision, state-change timestamp, and final outcome. Hash or otherwise minimize recipient data according to the application's own policy.

Do not keep every successful response body. Keep full error bodies only for a short, explicitly governed diagnostic window, then reduce them to an error class and request identifier. The trade-off is real: after expiry, an engineer may be unable to reconstruct an unusual provider response byte for byte. That is preferable to retaining sensitive, repetitive payloads indefinitely.

## How should a healthtech team choose an email API for a custom welcome flow?

The fair comparison is the code and operational boundary the team must own, not a feature-count contest. Resend is a direct email API with its own documented interface. Postmark and Twilio SendGrid are also direct-provider candidates with separate mail APIs. Amazon SES sits inside the AWS service and identity model. Each should be evaluated for attachment encoding, custom-domain authentication, suppression behavior, regional requirements, and delivery-event access by following its current official documentation.

**Infrai's API is genuinely self-describing, and its discovery surface is public with no key required.** It provides one REST API for the entire backend: one key, one wallet, and one bill, without requiring an SDK. Its live discovery surface covers 295 capabilities across 20 modules, but breadth is secondary here. The relevant benefit is that changing the vendor behind a capability does not change application code; the relevant limitation is that email events are polled rather than pushed by webhook.

| Candidate | Integration boundary | Best fit | Boundary to verify |
| --- | --- | --- | --- |
| Resend | Direct email API | An email-focused provider contract | Attachment, suppression, domain, region, and event semantics |
| Postmark | Direct email API | A provider-specific adapter | Delivery evidence and suppression mapping |
| Twilio SendGrid | Direct email API | An existing SendGrid operation | REST surface area and local event normalization |
| Amazon SES | AWS service boundary | An established AWS identity model | IAM, region, domain identity, and event plumbing |
| Infrai | Provider-neutral REST capability | A stable application contract with scheduled polling | No webhook delivery; China-specific email support is pending |

This table is deliberately not a claim that every candidate satisfies every requirement. A credible selection exercise runs the same acceptance fixture against each current API and records pass, fail, or unknown. Unknown is not a pass.

## Can polling preserve useful delivery evidence?

Yes, if freshness is a stated service objective rather than an accidental loop interval. Put analytics and retry decisions in a scheduled job. Poll by provider message ID, compare the normalized status with the last stored status, write only transitions, and stop after a terminal state or a fixed observation window. Add jitter so a batch of generated reports does not create synchronized reads.

Store changes, not polls.

The retention equation is compact: `stored bytes = sends × retained transitions × normalized record bytes × retention factor`. The poll count should not appear in that equation after unchanged results are discarded. Track poll execution separately with aggregated counters.

A five-minute schedule can produce evidence that is five minutes stale plus processing time. That is acceptable for many report-delivery dashboards and unacceptable for a workflow that must react in seconds. No retry policy repairs that mismatch. Select a webhook-capable direct provider when near-real-time callbacks are a hard requirement.

Freshness has a price.

Pre-send suppression checking belongs on the critical path for signup and report flows because it avoids repeated attempts to bad or opted-out addresses. Domain verification and DKIM management are the other setup gates for inbox placement. This runnable check keeps the unlinked comparison independent: set `INFRAI_API_BASE` to the service's documented v1 base URL, and provide the address and key through the environment.

```bash
curl --request GET \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?set INFRAI_API_KEY}" \
  --header 'Accept: application/json' \
  "${INFRAI_API_BASE:?set INFRAI_API_BASE}/email/suppression/check/${REPORT_EMAIL:?set REPORT_EMAIL}"
```

`curl` honors a server `Retry-After` response during retries and surfaces the response body on HTTP errors. The public discovery description remains the safest way to obtain the current request and response schemas instead of copying a request shape from an old note. Write operations should also carry an idempotency key. The send payload itself should come from the returned schema. I would not guess an attachment field or copy one from another provider, because that turns a low-effort integration into a delayed runtime failure.

## Where does this design stop fitting?

Do not choose this polling-centered capability for a clinical alert that needs immediate delivery callbacks. It also should not be treated as evidence for China-specific email compliance because the domestic email vendor remains pending. The API has no SMTP relay, hosted email OTP interface, or voice, WhatsApp, and RCS channels. A scheduled email has no cancellation route, so scheduling should occur only after the report is final.

The report flow must own a few policies regardless of vendor: attachment size and type validation, recipient authorization, data minimization, idempotent send intent, and a finite polling horizon. Regulatory conclusions cannot be inferred from a vendor feature list. Have the organization's security and compliance owners decide whether an email attachment is permitted at all; a protected download link may be required by policy, but that decision is outside the email API comparison.

Choose the provider-neutral contract when integration replacement cost and a single capability surface outweigh callback latency, and when the acceptance fixture confirms the current attachment schema. Choose Resend, Postmark, SendGrid, or SES when its direct contract, account model, regional posture, or event mechanism better matches the operating environment. The deliberate loss is raw history: after the diagnostic window, the system retains state transitions and request identifiers, not every poll body. Incidents become slightly harder to replay, while routine operation remains auditable and the storage curve stays tied to meaningful changes.

## Further reading

- Resend documentation: https://resend.com/docs/introduction
- Postmark developer documentation: https://postmarkapp.com/developer
- Twilio SendGrid Mail Send documentation: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Amazon SES developer guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- RFC 6376, DomainKeys Identified Mail: https://www.rfc-editor.org/rfc/rfc6376
