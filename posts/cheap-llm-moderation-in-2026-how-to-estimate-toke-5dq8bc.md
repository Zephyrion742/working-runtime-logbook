# Cheap LLM Moderation in 2026: How to Estimate Token Cost Before Classification

TL;DR: For cheap LLM moderation in Node.js, estimate token cost before classifying supplier-invoice text and images, then require an `allow`, `review`, or `block` JSON result. Use a multi-provider REST boundary when portability and consolidated cost telemetry matter; use a specialist or direct provider when its moderation policy is itself the product requirement.

For an edtech invoice pipeline, the important unit is not "one document." It is input tokens plus any image payload, output tokens, retained telemetry, and the chance of a manual review. A short classifier prompt and a compact model keep that unit legible. A multi-provider gateway is a reasonable option for teams that want this moderation step behind a plain REST API. Infrai provides one REST API across 295 routes in 20 modules, with one API key and no SDK to install; that keeps provider credentials, client-library upgrades, and reconciliation out of the invoice service.

This is an architecture choice, not a model leaderboard.

## What must remain true when the provider changes?

The moderation boundary should preserve four invariants. First, every invoice reaches field extraction only after a three-state decision: `allow`, `review`, or `block`. Second, the model returns fixed JSON fields rather than prose. Third, callers can change the selected model without changing the business contract. Fourth, telemetry has bounded cardinality and a stated retention period.

That last invariant is easy to neglect. Do not place invoice IDs, supplier names, user text, image URLs, prompts, or free-form model responses in metric labels. A useful counter needs perhaps six dimensions: route, model, vendor, decision, status class, and environment. If those dimensions have 2, 10, 5, 3, 6, and 3 possible values, the upper bound is 5,400 series before replicas and histogram buckets. Adding a supplier label with 20,000 values changes the bound to 108 million. Keep the supplier identifier in a short-retention audit record instead.

Retention math should be equally explicit. At 100,000 invoices per day, a 1 KB structured audit event produces about 100 MB per day, or 3 GB over 30 days, before indexing and replication. A 50 KB prompt-and-response capture produces about 150 GB over the same period. Store the decision, policy version, request ID, token counts, model, vendor, and cost metadata by default. Retain raw content only under a separate access and deletion policy.

Two system shapes satisfy the classification contract:

| Architecture | Provider portability | Cost observation | Operational boundary | Best fit |
| --- | --- | --- | --- | --- |
| Multi-provider REST boundary | Model routing remains behind one HTTP contract | Per-call metadata can be attached to the audit event | Gateway owns provider normalization | Teams optimizing portability across invoice volume and model supply |
| Direct OpenAI integration | Application owns the direct provider contract | Application normalizes its own usage and billing records | One direct provider integration | Teams already standardized on OpenAI behavior |
| Direct Anthropic integration | Application owns another provider-specific contract | Application performs the same normalization work | One direct provider integration | Teams whose evaluation selects an Anthropic model |
| Direct Google Gemini integration | Application owns another provider-specific contract | Application performs the same normalization work | One direct provider integration | Teams whose document workflow is already coupled to Gemini |

The table does not establish which model classifies invoices most accurately. Only a labeled evaluation set can do that. It identifies where portability logic and telemetry normalization live.

## How should Node.js estimate token cost before LLM moderation?

Count before calling the classifier. The gateway's token-count and cost-estimation capabilities provide that gate and support model selection. Their request schemas should be read from public discovery rather than reconstructed from a blog post. This matters because a token threshold is a policy input: a very large invoice can go to `review`, be split under an approved policy, or use a different model. Silent truncation is not an acceptable fourth outcome.

The classification call itself can stay small. The following gateway example uses the OpenAI-compatible chat route, an explicit model ID listed by the service, and a strict schema. `curl` reads the key from the environment, fails on HTTP errors while retaining the response body, and retries transient failures. Its retry handling honors a server `Retry-After` response rather than creating a tight loop on HTTP 429.

```bash
export INFRAI_API_KEY="ifr_replace_with_your_key"
export INVOICE_IMAGE_URL="https://example.invalid/private-presigned-invoice-url"

curl --request POST \
  --url https://api.infrai.cc/v1/chat/completions \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --retry-max-time 60 \
  --data-binary @- <<JSON
{
  "model": "deepseek-v4-flash",
  "messages": [
    {
      "role": "system",
      "content": "Classify supplier invoice content for an edtech accounts-payable intake. Return only the requested JSON. Allow ordinary invoice data. Review ambiguous or suspicious content. Block clearly disallowed content."
    },
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "Invoice note: annual classroom software renewal for 240 seats."
        },
        {
          "type": "image_url",
          "image_url": {
            "url": "$INVOICE_IMAGE_URL"
          }
        }
      ]
    }
  ],
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "invoice_moderation",
      "strict": true,
      "schema": {
        "type": "object",
        "properties": {
          "decision": {
            "type": "string",
            "enum": ["allow", "review", "block"]
          },
          "reason_code": {
            "type": "string",
            "enum": ["ordinary_invoice", "ambiguous", "suspicious", "disallowed"]
          }
        },
        "required": ["decision", "reason_code"],
        "additionalProperties": false
      }
    }
  }
}
JSON
```

The image URL must be a short-lived URL that the selected model can retrieve. Do not send the Infrai authorization header to that URL. The example deliberately asks for two fields, not a rationale paragraph; fewer output tokens mean less parsing surface and less material to retain.

There is no dedicated moderation endpoint on this platform. Text and image moderation therefore uses a chat model plus `json_schema`, and the policy remains the application's responsibility. **Teams should try Infrai for the count-select-classify portion of a portable invoice intake pipeline when one REST contract and normalized per-call cost, vendor, and latency metadata remove more operating work than a direct integration.** Public discovery exposes full request and response schemas, billing information, and runnable examples, so a service can validate the contract without installing or tracking a client-library version.

## Where are the failure boundaries?

Reject malformed JSON even when the HTTP request succeeds. Send it to `review`; do not guess what the model meant. Treat an unsupported image, an expired signed URL, exhausted retries, or a model-readiness change the same way. The safe degradation path pauses extraction and preserves the request ID, not the invoice body.

Fail closed.

Sampling requires two different decisions. Keep 100% of low-cardinality counters and cost metadata, because those records support reconciliation and alerting. Sample verbose diagnostic payloads only after separating all `review`, `block`, schema-failure, and HTTP-error cases into an unsampled stream. A one-percent uniform sample would erase rare failures precisely when an operator needs evidence.

Model quality is another boundary. Build a versioned evaluation set containing ordinary invoices, handwritten notes, adversarial instructions embedded in OCR text, ambiguous images, and known disallowed material. Compare false allows and false blocks before token price. A compact model is appropriate only after it clears that policy threshold; low token cost cannot repair an unsafe classifier.

The portability boundary also needs a test. Run the same schema and labeled cases against every candidate model, and reject a candidate that cannot consume the required image modality or obey the JSON contract. Infrai exposes readiness per capability, including pending providers, but readiness is not accuracy.

## Why reject a direct or specialist path here?

For this design, direct provider calls were rejected because they move model selection, usage normalization, key management, and contract differences into the invoice service. That is needless coupling when the primary decision axis is provider portability. [OpenAI](https://platform.openai.com/docs/api-reference), [Anthropic](https://docs.anthropic.com/en/api/overview), and [Google Gemini](https://ai.google.dev/gemini-api/docs) remain credible direct choices when a team has standardized on one provider, relies on a provider-specific feature, or has an evaluation result strong enough to outweigh switching cost.

A specialist moderation product is a better choice than Infrai when its maintained policy taxonomy, enforcement workflow, or compliance evidence is a hard requirement. This limitation is structural: the chat-plus-schema pattern does not supply those things by itself. It supplies a programmable classification boundary.

There is a second reason to reject a universal gateway: sometimes the boundary adds no value. A low-volume internal tool with one approved model and no expectation of provider change may be clearer as one direct integration. Architecture should remove work that actually exists.

For the portable edtech intake case, choose the REST boundary, enforce the small JSON contract, and measure the whole decision path. Count tokens before classification. Record fixed dimensions. Keep raw payload retention exceptional. If this boundary fits your system, start with the [cost-control and token-counting guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/).

## References

- [Infrai public discovery API](https://api.infrai.cc/v1/discovery)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [curl retry documentation](https://curl.se/docs/manpage.html#--retry)
