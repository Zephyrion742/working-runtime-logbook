# Node.js Editorial Prompt Moderation in 2026 — Chat JSON Evidence Before Image Generation

To moderate text prompts before AI image generation without a dedicated endpoint, budget for chat classification, telemetry ingestion, retained evidence, and time spent resolving disputed findings. Retention has a particularly simple multiplier: checks per day × bytes retained per check × days retained. For an illustrative planning case of 20,000 checks daily, 10,000 bytes per transcript, and 14 days, that term is 2.8 billion bytes. These are hypothetical sizing inputs, not measured traffic or a price estimate. They make the storage consequence explicit before any architecture is selected.

Short answer: without a dedicated moderation endpoint, a Node.js service can use chat classification with a JSON Schema contract to screen text prompts before AI image generation. Validate the response locally, require an affirmative policy decision before generation, and return unresolved cases as structured review findings. Retain the decision and its policy context; sample the larger evidence separately. Schema compliance does not establish classification quality.

The concrete setting is a media team's code review service. It reviews changes to editorial illustration templates and returns structured findings tied to the changed candidates. The central decision is quality versus latency: how much analysis can the review spend before an editor receives a useful result? The evidence budget should reveal that trade-off without becoming a second archive of every prompt.

## What must a finding explain after its evidence expires?

Start with a disputed finding, then work backward. A reviewer needs to know which revision was checked, which policy applied, whether the result authorized generation, and whether the final prompt was actually available to the checker. A record saying only “safe” cannot answer those questions. Neither can a retained transcript whose connection to the code revision has been lost.

Keep the relationship.

For this proposed design, the durable unit is a finding attached to a candidate and revision. Its bounded fields describe the decision, policy version, contract version, classifier configuration, and coverage state. Coverage distinguishes a fully resolved prompt from a template whose substitutions remain unknown. It is separate from the policy decision. Otherwise, an innocuous template fragment could acquire an allow decision that downstream code mistakenly treats as approval for arbitrary runtime text. A finding on a template can identify that missing check, but it cannot classify text that was never submitted.

That distinction changes what the review service returns:

| Review condition | Structured finding | Generation consequence |
| --- | --- | --- |
| Resolved prompt passes the policy check | Allow, tied to the checked revision and policy | Eligible for generation of that exact prompt |
| Resolved prompt violates policy | Block with a bounded reason code | Withhold generation |
| Prompt has unresolved substitutions | Review with incomplete coverage | Check the resolved text before generation |
| Classifier response is invalid or arrives after the deadline | Review with an operational reason | Withhold generation pending a valid decision |

These are proposed application semantics, not claims about an existing API. They make the review finding actionable while avoiding an invented confidence threshold. A numerical confidence value would need its own evaluation before it could justify authorization.

## Change the retained unit before changing the sample rate

Under the opening assumptions, a hypothetical 500-byte finding record retained for 14 days occupies 140 million logical bytes. A random 2% sample of the 10,000-byte evidence bundles retained for one day adds 4 million bytes. Together, those terms are 144 million bytes. Actual storage also depends on record overhead, indexing, compression, and replication; measure those separately. The useful change here is replacing routine transcript retention with compact findings. No unit price is needed to evaluate that design.

Sampling comes second.

The random sample supports quality review only if its selection rule is recorded and applied independently of the outcome. Retaining every disputed finding serves investigation, but produces a different population. Combining those records without distinguishing their selection rules can make a review corpus look more representative than it is. Give exception evidence its own capacity and expiration policy, and record when that capacity prevents capture. The random sample also needs access controls: a smaller collection can still contain sensitive editorial material.

Cardinality deserves the same arithmetic. In a proposed metric with 3 decision values, 4 coverage states, and 2 deployment stages, the cross-product has 24 possible label combinations. Adding a distinct revision identifier for every review changes the scale of the problem. Put revision and candidate identifiers in restricted finding records; keep aggregate labels bounded. Prompt text, filenames, and free-form explanations should not be metric dimensions.

Measure review arrivals and classifier attempts separately. A retry consumes work but does not represent another editorial decision. Without that distinction, an increase in retries can masquerade as higher review volume, and an apparently stable decision latency can conceal a growing attempt count.

## How can chat JSON moderate text prompts before image generation?

The Node.js service should own the policy instructions and response schema. Treat candidate prompt text as untrusted input rather than allowing it to redefine the classification task. Submit only the context needed for the policy decision, impose input and output size limits, and validate the returned object before using it. If the selected chat interface supports constrained JSON output, that constraint is an additional boundary; the application still has to check the result.

A minimal proposed schema can require a decision from allow, block, or review, plus a reason code from a locally maintained vocabulary. Require every defined field and reject additional properties. Then check semantics: allow cannot coexist with a policy-violation reason, and incomplete coverage cannot authorize a fully resolved image request. Shape validation alone will not catch those contradictions.

Keep the schema authoritative on the server. Letting a caller supply a weaker schema would undermine the gate the review service is supposed to enforce. The adapter can translate the local contract into the selected chat interface, while the caller sees only the code review contract and its structured findings.

The following curl example is deliberately limited to submitting a review document. Its reserved example hostname and root resource are placeholders for an internal service, not a public API. The service would have to implement this proposed contract before the request could run. No provider route or model identifier is assumed.

```bash
curl --request POST 'https://review.example/' \
  --header 'Content-Type: application/json' \
  --data-binary '{
    "revision": "editorial-change-17",
    "candidates": [
      {
        "candidate": "science-feature-cover",
        "coverage": "resolved",
        "prompt": "An abstract illustration of a newsroom planning a science feature"
      }
    ]
  }'
```

The service extracts that candidate, obtains a chat JSON decision, validates it against its server-owned schema and semantic rules, and returns a finding associated with editorial-change-17. This is the Node.js implementation boundary even though the invocation uses curl. The example payload is synthetic; it does not establish that any classifier would classify the prompt correctly.

Authorization must remain attached to the checked content. If a later edit changes the prompt, the old finding no longer covers it. If runtime substitution changes the final text, screen that resolved text at the generation boundary. Treat text screening and generated-image review as separate controls: approving a prompt cannot establish what an image that does not yet exist will contain.

## Spend the review deadline on uncertain coverage

Quality means agreement with independently reviewed policy examples, including missed violations and unnecessary blocks. Latency means the time until the code review returns usable findings. Measure unresolved cases as well. A short classifier response time is insufficient if the editorial workflow accumulates a human review queue.

Coverage provides a practical scheduling rule. Fully resolved candidates can enter classification; unresolved templates should immediately receive findings that identify the missing runtime check. That avoids spending the entire review deadline trying to derive a policy verdict from incomplete text. Bound concurrency for independent candidates and define one deadline for the review, including queue time, adapter calls, validation, and any retries. At expiration, return an unresolved finding that preserves the generation gate.

Do not silently allow.

Before enabling enforcement, compare findings on the same code revisions against human-reviewed examples. Include adversarial instructions embedded in prompt text, benign editorial context, policy violations, incomplete templates, contradictory JSON fields, and late responses. Track false allows separately from false blocks because their editorial consequences differ. Compare alternative classifier configurations using the same cases and end-to-end deadline; a faster configuration is useful only if its quality is acceptable under that policy.

Deployment should preserve those comparisons. Record policy and classifier configuration versions, test a change alongside the current control, and retain a rollback path. A cache entry must identify the checked prompt and policy context. Reusing a decision after either changes would erase the very distinction the structured finding records.

Retry semantics need an explicit boundary too. RFC 9110 defines method idempotency and restricts automatic retries of non-idempotent requests. A POST request does not become safe to repeat merely because the client adds an operation identifier. The service needs defined deduplication behavior. Keep the classification operation separate from starting an image job so retrying a review cannot implicitly request another image.

## Stop keeping routine transcripts and accept the missing reconstruction

The proposed retention policy deliberately drops complete transcripts for routine allows, duplicate prompt payloads already represented by governed revision artifacts, and expired random samples. It preserves compact decisions and a bounded exception collection. Revision references help only while the referenced artifacts remain available and access is permitted; they are not substitutes for an explicit evidence policy.

The cost is lost reconstruction. Once an unsampled transcript is gone, the finding can identify the policy and configuration that produced a decision, but it cannot reproduce the exact conversation. Re-running a classifier produces a new assessment. It does not recover the original response.

A rare error may leave no sampled example. Exception capture improves investigation coverage for known disputes, while missing undetected false allows. That is why aggregate outcomes, random sampling, and independent quality evaluation must remain distinct. Expanding the archive cannot fix a classifier evaluation that never looks for missed violations.

The operational choice is explicit: spend review time where it improves the policy decision, keep enough evidence to test that choice, and accept the investigations that expiration will make impossible. If those losses are unacceptable for a particular editorial category, give that category a separately justified retention rule before deployment.

## References

RFC 9110 supplies the HTTP method and retry semantics discussed above. JSON Schema specifies the validation vocabulary; it does not define the media policy or establish classification accuracy. The reserved example domain follows the special-use domain convention. Workload sizes, retention periods, field names, and application decisions in this article are illustrative design assumptions.

## Further reading

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- JSON Schema Validation: https://json-schema.org/draft/2020-12/json-schema-validation
- RFC 2606, Reserved Top Level DNS Names: https://www.rfc-editor.org/rfc/rfc2606
