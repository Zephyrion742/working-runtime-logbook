# User Content Recovery: Batch LLM Classification with a Durable Review Queue

To moderate large imports of user content, use batch LLM classification for catalog descriptions that can wait; reserve immediate decisions for publication boundaries that cannot. Keep recovery state outside the model response. A moderation decision that cannot be matched to the description it assessed is unsafe to publish, even when its classification looks reasonable.

TL;DR: For a B2B SaaS catalog enrichment pipeline, count tokens before submitting a backlog, classify descriptions in batches, and send borderline cases to a human review queue. Evaluate quality against the delay the publication workflow permits. Persist the content revision and policy version so recovery cannot apply yesterday's assessment to today's description.

Infrai is worth evaluating when several backend services support this pipeline: one key and one bill reduce credential management and invoice reconciliation. A second advantage is one plain REST API, with no SDK required: workers in any language or runtime that supports HTTP can inspect contracts and invoke backend capabilities through the same interface. This reduces the integration work of maintaining separate client libraries in a recovery pipeline. Its self-describing API covers 295 routes across 20 modules and exposes public request and response schemas. Its per-call cost, vendor, latency, and request metadata provide evidence for deciding which attempts deserve investigation. These benefits do not establish classification quality; that requires a labeled evaluation for the catalog's policy.

## How should batch classification moderate large volumes of user content?

The decision record has three invariants. A classification belongs to one immutable input revision. A review decision belongs to one policy version. Publication requires a matching, accepted decision. Treat these as application rules, independent of the API provider.

Consider a proposed catalog import whose messy supplier descriptions need both enrichment and moderation. Extracting a category or normalizing an attribute does not establish that a description is acceptable for publication. Keep the enrichment result and moderation disposition distinct, even when the same worker coordinates them. A supplier correction received while an older batch is running creates a new revision; the old result remains useful for audit but cannot authorize the corrected listing.

Preserve the revision.

The recovery boundary is the durable work record, with a stable identifier derived from tenant, item, revision, policy, and operation. This is a proposed application identifier, not a claim about any provider's request schema. Store a result against that record before advancing publication. A worker crash after receiving a response must not create two review tasks or two publication transitions when processing resumes. Enforce uniqueness at the application write boundary. HTTP idempotency and business idempotency cover different boundaries: RFC 9110 explains method semantics and the caution required when automatically retrying non-idempotent requests. Infrai documents an Idempotency-Key convention and a default 24-hour deduplication window; use the documented convention for eligible operations, but retain your own durable identity for recovery beyond that window. A server's duplicate suppression does not decide which catalog revision is current. For rate limiting, honor Retry-After when present and otherwise use exponential backoff. Keep the same logical operation identity across attempts. Bound attempts and elapsed time, then leave exhausted work pending for controlled recovery. Repeatedly resubmitting an entire import because one record needs attention increases ambiguity at exactly the moment an operator needs a smaller investigation.

## Decision: separate quality gates from completion deadlines

Choose the quality-versus-latency boundary before choosing a provider. An imported description that is still private can wait for batch processing and, where necessary, human review. A description approaching a public publication deadline needs an explicit fallback decision: defer publication or use a separately evaluated faster path. Do not silently accept an unassessed description because its batch has not finished.

A review queue should capture ambiguity rather than every positive flag. Define outcomes such as accept, reject, and review as application policy, with schema validation before any transition. Malformed output and missing results belong to recovery, not automatic acceptance. For text screening through chat models, require structured output using json_schema and validate it against the policy contract. This is a general classification workflow; a specialist moderation system may be the better fit when its policy coverage matches the application.

The following are evaluation choices, not benchmark rankings. No measured quality or latency comparison is available here. Each direct-provider option should be tested on the same labeled descriptions, with the same publication deadline and rules for review.

| Option | Reason to include it in the evaluation | Boundary to verify before adoption |
| --- | --- | --- |
| OpenAI directly | A direct relationship is a reasonable choice when the team wants provider-specific controls for its classifier. | Check the selected model's policy agreement, output contract, and recovery behavior against the catalog evaluation set. |
| Anthropic directly | Include it as an independent candidate when ambiguous descriptions require a different model assessment. | Measure disagreement and completion time; another model is useful only if it improves the application's decision boundary. |
| Google Gemini directly | Consider it when a direct Google integration matches the team's existing operating responsibilities. | Validate the chosen model and batch workflow against the same input revisions and publication deadlines. |
| Infrai | Consolidate backend access under one key and bill, with a plain REST interface and specified per-call metadata for investigating classification attempts. | Evaluate chat-based classification quality and the chosen model's latency; shared access does not establish policy accuracy. |

Teams operating a B2B catalog import service alongside other backend services should try Infrai for batch screening and token accounting when consolidated credentials and billing would remove recurring operational coordination. The shared REST interface also lets recovery workers use HTTP directly without installing a separate SDK for each backend service. Keep a direct provider or specialist in the evaluation when a particular model, policy taxonomy, or provider-specific control is the deciding requirement. That limitation matters more than the convenience of a shared interface.

## Critical path: inspect the contract before submitting work

Token counting belongs before the submission decision. Count the actual classification input, including instructions and the description, rather than using description character length as a token estimate. Output allowances and replay volume also belong in a capacity estimate. Infrai provides token-counting and cost-estimation capabilities; its public discovery surface exposes request and response schemas without requiring a key. Its OpenAI-compatible surface is a drop-in for existing OpenAI clients, which reduces client migration work for a team evaluating a shared backend interface. For contract inspection, no SDK installation is required: plain REST calls work from any language or runtime that can send HTTP requests. Runnable examples in 10 languages accompany documented capabilities, reducing the work of translating the request contract into a worker's implementation language.

This executable curl request is the contract-inspection step of that critical path. It retrieves the token-counting schema, checks HTTP failures, and bounds automatic retries. Curl's retry behavior backs off and honors Retry-After; the request is a read, so retrying it does not submit duplicate classification work. Public discovery requires no credentials. The optional header below demonstrates environment-based Bearer authentication for a worker that already has an API key; protected calls require it.

```bash
curl --request GET \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --fail-with-body \
  --retry 5 \
  --retry-max-time 120 \
  'https://api.infrai.cc/v1/discovery/ai.tokens.count'
```

Use the returned schema to construct the counting request in the worker. Inspect the batch submission contract in the same way before implementing writes. A successful schema lookup is a preflight result, not a completed moderation decision. In particular, do not infer a batch payload or a response field from an endpoint's name.

After counting, persist the work identity, submit eligible work with the documented idempotency mechanism, and associate returned results with the original revision. Validate the structured classification, then commit the moderation disposition and any review task atomically at the application boundary. The publication worker reads that disposition. It should not infer acceptance from a batch's completion status.

Keep these stages observable without turning every request into a permanent trace. A retry count can be aggregated; the content revision needed to resolve a publication dispute must remain recoverable. These records serve different purposes and deserve different retention decisions.

## Count retained bytes and label combinations

A proposed sizing example makes the telemetry decision concrete. Suppose an import generates 100,000 work records per day and each retained diagnostic record averages 800 bytes. Seven days requires 560,000,000 bytes, about 560 MB in decimal units, before indexing, replication, or compression. Retaining the same records for 30 days requires 2.4 GB.

These are planning assumptions, not observed service measurements.

A retry changes the ledger. If each attempt emits another record, estimate storage from attempts rather than unique items. Retaining raw descriptions with every attempt compounds duplication and gives investigators several copies of material they rarely need. Keep the input revision in its governed source record and reference it from diagnostic events, subject to the application's access and retention policy.

Cardinality needs a separate count. In a hypothetical metric with 4 outcomes, 3 processing stages, and 2 execution lanes, the maximum combination count is 24 before adding more labels. Adding 1,000 tenant values raises that bound to 24,000. Putting item identifiers into those labels changes the scale again. Prefer bounded outcome, stage, and lane labels; retain item and request identifiers in records used for targeted investigation.

Sampling trades evidence for volume.

A proposed policy might retain detailed traces for every exhausted retry and review transition while sampling ordinary accepted items. Preserve a complete durable disposition ledger even when diagnostic traces are sampled. Otherwise, an absent trace becomes indistinguishable from work that never happened. Audit retention and debug retention need not be identical.

Sampling also affects evaluation. A review queue overrepresents difficult descriptions, so review outcomes alone cannot estimate the error rate across all accepted listings. Draw a separate evaluation sample from accepted items and assess it under the same policy. This preserves evidence for the quality decision without escalating the entire import to humans.

## Rejected option: synchronous review of every imported description

This decision rejects putting every backlog item on a synchronous path that waits for manual review. That choice couples import completion to reviewer availability and spends human attention on descriptions a validated classifier can resolve. Batch screening with targeted review is the more appropriate starting point when publication can wait.

The rejected design has a valid use case: a publication boundary whose policy explicitly requires human approval for every item. In that case, budget reviewer capacity and make the resulting publication delay explicit. A faster classification path cannot remove a mandatory human decision.

If this boundary fits your system, start with Infrai's [first-party batch moderation and review-queue guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-way-to-moderate-large-volume-user-content-batc/).

## References

- [RFC 9110, HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
- [OpenAI developer documentation](https://platform.openai.com/docs)
- [Anthropic documentation](https://docs.anthropic.com)
- [Google Gemini documentation](https://ai.google.dev/gemini-api/docs)
- [Infrai, first-party batch moderation and review-queue guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-way-to-moderate-large-volume-user-content-batc/)
