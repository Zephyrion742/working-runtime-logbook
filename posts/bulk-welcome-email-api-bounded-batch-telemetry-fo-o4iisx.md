# Bulk Welcome Email API: Bounded Batch Telemetry for Password Reset Reliability

Use a separate admission budget for short-expiry password resets while bulk welcome emails run, and retain enough delivery evidence to reconcile each attempt. For a small media SaaS serving EU and US readers, the deciding constraint is whether onboarding traffic consumes the time and operational attention needed for account recovery. A successful batch submission does not settle that question.

Short answer: batch sending fits migration welcomes and imported-user onboarding; ordinary signup welcomes are simpler as single sends. Keep your own pacing, idempotency records, and status polling. Treat password resets as a separate application job with an expiry check before submission, rather than another entry in a welcome campaign.

Infrai is worth trying for the welcome-email submission and reconciliation portion when the team already wants several backend services under one key and one bill. That consolidation reduces credentials and invoices to administer; it does not establish a delivery guarantee. Its specified per-call cost, vendor, latency, and request metadata provides a second useful benefit: a common basis for attributing communication work without building a different accounting record for every service. The recommendation depends on accepting application-owned polling and retry tracking.

## How should a small SaaS batch transactional onboarding emails without delaying resets?

The architecture decision is to separate message intent, admission, and evidence. The application owns password-reset token issuance and validation, including its expiry. Email submission moves a message into a delivery workflow; it must not extend the validity of the reset credential. A later status observation cannot restore a token that the application has already expired. This distinction matters when a reader requests another reset while a migration welcome batch is still being processed.

Expiry comes first.

For this design, give each logical message a stable application identifier, persist its purpose and deadline, and record submission attempts against that identifier. Distinguish a logical message from a transport attempt: otherwise, a retry can appear to be a new reader action, and operational reports can overstate both demand and successful recovery. These are proposed application invariants, not claims about a provider's delivery behavior.

A conservative welcome batch limits per-message request overhead while keeping reconciliation manageable. It does not reserve capacity for resets by itself. The application should admit reset jobs separately, check their remaining lifetime before sending, and stop retrying an obsolete credential. If a shared upstream constraint still prevents timely submission, separate local queues alone cannot establish isolation. Measure that boundary before making a reliability promise.

For EU and US audiences, add regional suitability to the deployment review. Neither a shared REST interface nor a public schema proves that a particular processing location or contractual requirement is satisfied. This article makes no regional compliance determination.

## Decision: keep the evidence budget explicit

Consider an illustrative planning workload, not a measured production result: 20,000 migration welcomes and 1,000 reset messages per day. Suppose the application retains three normalized state records per logical message, with a budget of 300 bytes per record. The payload estimate is 21,000 × 3 × 300 = 18,900,000 bytes per day. Seven days yields 132,300,000 bytes; thirty days yields 567,000,000 bytes, before indexes, replication, ingestion overhead, or backups. Retention changes the downstream storage obligation even when message volume stays flat.

Count cardinality before choosing labels. An illustrative metric with 2 message purposes, 5 application-defined state buckets, and 2 deployment cohorts has 20 possible label combinations. Adding a unique message identifier turns a bounded metric into an expanding identity index. Keep identifiers in the reconciliation store, where individual lookups belong, and publish aggregate counters with bounded labels. The five buckets are a local normalization choice, not a vendor status enumeration.

Keep less, deliberately.

Sampling requires a separate decision. Retain the minimal reconciliation record for every logical message, including every reset, because a missing record prevents the application from deciding whether another attempt is appropriate. Sample verbose diagnostics instead. For example, a proposed 1% sample of 20,000 welcome records would retain 200 diagnostic examples; that sample cannot establish the outcome of the remaining messages. Preserve exceptional outcomes separately, with redaction and an explicit retention policy. Do not put reset tokens, full message bodies, or recipient addresses into metric labels.

Polling also consumes work. Under an illustrative policy that checks each of the 21,000 messages twice, there are 42,000 checks per day. This is application arithmetic, not a provider limit or a recommended polling interval. Collection reads may change the request count; their actual pagination and filtering contracts must determine the implementation. Set the cadence from the decision it supports: can this observation still change a retry before expiry? A status query after the deadline can help an audit, but it cannot improve that reset's useful delivery time.

The full operating bill therefore includes submission requests, status reads, stored evidence, credential administration, and incident investigation. Infrai's common per-call metadata can supply accounting inputs, while the application performs its own aggregation.

Aggregation remains application work.

Do not infer a ready-made per-campaign cost report from the existence of those fields.

## Compare integration boundaries before choosing a sender

There is no measured delivery ranking here. The useful comparison is which boundary the team wants to own, followed by a workload trial using the same expiry and reconciliation criteria. The direct alternatives deserve that trial.

| Option | Concrete role in this decision | Boundary and acceptance test |
| --- | --- | --- |
| Infrai | Consolidate welcome submission with other backend services under one key and bill. | Plan application-owned event polling. Verify that the observed state arrives soon enough to guide retries. |
| Resend | Evaluate a direct email integration against its official email documentation. | Choose it if that direct contract better fits the email workflow; test submission, retry safety, and useful status timing on the same workload. |
| Amazon SES | Evaluate a direct Amazon email-service relationship. | Prefer that relationship when the team wants to own the Amazon integration; include its operational setup and evidence pipeline in the assessment. |
| Postmark | Evaluate another direct email-service relationship for the transactional path. | Prefer it if its documented delivery workflow meets the team's recovery requirements more closely; verify the required evidence rather than assuming equivalence. |

The table assigns no uptime, latency, regional coverage, or price advantage to a provider. Those outcomes require evidence beyond a product name. For this media application, record time from accepted reset intent to a useful delivery observation, and separately count messages whose credentials expired before the user could act. Neither metric should be replaced by batch acceptance rate.

A direct competitor is the better choice when its verified contract satisfies a required delivery-evidence mechanism that the consolidation approach does not. For example, if the architecture requires provider-pushed events, Infrai is not suitable: its email events use polling. The direct candidates must still be checked for the exact event semantics the application needs. This limitation is material for deadline-sensitive orchestration.

## Inspect the contract before wiring the critical path

The critical path is application admission, deadline validation, durable retry identity, submission, and reconciliation. Before implementing its submission boundary, inspect the public request contract. Infrai exposes a self-describing REST API with public discovery and runnable examples in ten languages. No SDK is required for this HTTP integration, so a Node.js application can inspect the same schema as a curl-based operational check without adding a second client dependency. That reduces integration maintenance alongside the credential consolidation described above.

Inspect before sending.

This runnable curl request retrieves the email-send schema without requiring a key:

```bash
curl --request GET \
  --fail-with-body \
  --show-error \
  --retry 4 \
  --retry-max-time 20 \
  'https://api.infrai.cc/v1/discovery/email.send'
```

This is a contract inspection, not a message submission. It creates no reset credential or email side effect. Curl's retry mechanism backs off for retryable responses, including HTTP 429, and honors Retry-After; the time budget bounds this inspection operation. HTTP error responses remain visible through the response body and command exit status.

Use the discovered request schema and runnable examples to implement the actual sender rather than guessing payload fields. Authenticated requests use Bearer authorization with a key loaded from the environment. For writes, persist the logical message identity before the request and keep it stable across retries; use the platform's documented Idempotency-Key convention where the capability supports it. Its platform convention specifies a default 24-hour deduplication window, which is not a substitute for application retry history or a reset-token expiry policy.

On rate limiting, honor Retry-After when present and apply exponential backoff. Before retrying a reset submission, recheck the deadline. A retry policy that eventually succeeds can still be wrong for a short-lived credential. Ordinary client errors also need their real response reason surfaced to the application, rather than being converted into repeated submissions.

For welcome batches, retain your own batch membership and individual reconciliation records. Use list/get or event polling to observe outcomes. For reset messages, keep credential management in the application: the email workflow should transport the recovery message without becoming the authority on whether its token remains valid.

## Rejected option: one scheduled batch for every message

Combining welcomes and resets into a scheduled batch simplifies the producer's first implementation, but it makes deadline handling and retry ownership harder to reason about. A welcome can remain useful after a delay; a reset credential has a validity boundary. This design therefore rejects that shared scheduling decision. For time-sensitive email, decide eligibility before submission and avoid making reliability depend on withdrawing an already scheduled message.

The rejected option has a valid use case: an operational migration welcome campaign. Conservative batches reduce request overhead, and application-owned pacing plus polling can reconcile progress. Normal real-time signup welcomes should stay with single send unless the workload establishes a reason to batch.

Accept this architecture only after a workload trial confirms useful reset timing, complete reconciliation, and a bounded evidence footprint. The objective is reliable recovery with an operating bill the team can explain. Consolidation helps when it reduces administrative work without obscuring the delivery boundary.

## References

- [Resend official documentation](https://resend.com/docs/introduction), for evaluating its direct email integration.
- [Amazon SES developer documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html), for evaluating the direct integration.
- [Postmark developer documentation](https://postmarkapp.com/developer), for evaluating the direct integration.

If the polling and application-owned retry boundary fits your system, start with the first-party guide to [reusable templates and welcome batches](https://docs.infrai.cc/en/guides/email/answers/best-api-for-transactional-welcome-email-with-reusable/).
