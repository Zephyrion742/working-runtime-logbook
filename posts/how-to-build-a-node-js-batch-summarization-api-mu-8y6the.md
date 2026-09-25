# How to Build a Node.js Batch Summarization API: Multiple Documents

Short answer: for Node.js summarization across multiple documents, submit one asynchronous batch API job, hold the prompt constant, and make telemetry part of the quality gate. This is the practical default when a B2B SaaS catalog must enrich many records. A synchronous loop couples request latency to corpus size; a batch lets the backend choose quality deliberately and preserve evidence about the work that consumed tokens. The central trade-off is quality versus latency, not price.

A useful acceptance rule is concrete: every item returns the same parseable fields, every completed batch exposes its results, and every processing exception leaves a searchable telemetry record. Fast but inconsistent summaries do not pass. Neither do polished summaries disconnected from their processing record.

## Should a Node.js API summarize multiple documents as one batch?

A catalog import has two clocks. The user-facing clock wants an acknowledgment quickly. The enrichment clock may need to process thousands of descriptions, retry transient conditions, and preserve a stable output shape. Running one summarization call after another inside an HTTP handler merges those clocks. One slow document then extends the whole request, while a client retry can repeat completed work.

Queue the collection instead. Keep one summarization instruction for every item, including required fields and a null policy. That constraint reduces output-shape variance and makes downstream validation tractable. It also makes quality comparisons meaningful: if prompts differ between records, model behavior and prompt behavior become confounded.

Count before retaining. For 12,000 records, a metric label containing `product_id` can create 12,000 values; a bounded label such as `result=accepted|rejected` creates two. Put identifiers in logs or batch output, not metric labels. Retaining a 2 KB prompt and response for all 12,000 records stores about 24 MB before indexing and replicas. A 1% diagnostic sample stores about 240 KB. These are arithmetic examples, not measured platform costs.

Keep the exceptions. Sample routine successes aggressively, but retain every schema rejection and processing exception for the investigation window. This asymmetry is intentional: success volume estimates throughput, while exceptions explain wasted work.

That is the boundary.

## Derive the contract before writing the worker

Infrai is one option when a team values a self-describing REST surface. Its public discovery response reports 295 capabilities across 20 modules; each capability description supplies the HTTP path, full request and response JSON Schema, billing metadata, and runnable examples. A worker can read the live contract instead of learning another SDK. More important for this pipeline, AI runtime and telemetry share one key and request context. The token count and the exception that consumed it can be joined without an application-specific correlation identifier.

Inspect the live schemas first and save their generated curl examples. Then set the two JSON bodies from those schemas. This avoids copying undocumented fields into production. Both write calls below use the same environment-held credential and base URL. `BATCH_BODY` contains the fixed prompt and document items; `LOG_BODY` contains the returned platform request identifier plus the exception record.

```bash
set -euo pipefail
: "${INFRAI_API_KEY:?Set INFRAI_API_KEY}"
: "${INFRAI_BASE_URL:?Set INFRAI_BASE_URL to the documented API base}"
: "${BATCH_BODY:?Set BATCH_BODY from the discovery schema}"
: "${LOG_BODY:?Set LOG_BODY with the returned request identifier}"

curl --silent --show-error \
  --request POST \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: catalog-import-2026-09-26" \
  --data-binary "$BATCH_BODY" \
  "$INFRAI_BASE_URL/v1/ai/batch/submit"

curl --silent --show-error \
  --request POST \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --data-binary "$LOG_BODY" \
  "$INFRAI_BASE_URL/v1/logs/ingest"
```

The handoff is the request identifier returned by the first capability and placed into the second capability's schema-defined record. Keep the authorization value in an environment variable. For a write retry, retain the same idempotency key. On HTTP 429, honor `Retry-After` when present and otherwise apply exponential backoff; for another non-success status, surface the response body to the operator.

After submission, poll the documented status operation at a restrained interval. Fetch results only when processing completes, or request a downloadable export for an admin workflow. Store the batch identifier and state transition, not every poll response forever. One lifecycle is enough.

## Set a retention budget before sampling

Observability volume is a design input. Suppose a run has 12,000 items and emits six 700-byte informational events per item. Raw event payloads total about 50.4 MB per run: `12,000 x 6 x 700`. At four runs per day and 30 days of retention, that is roughly 6.05 GB before index overhead or replication. The exact bill depends on the telemetry backend, but the byte count already shows where to intervene.

Keep one batch-level start event, one completion event, every exception, and a small deterministic sample of item-level successes. Sampling by a stable record hash keeps the same products visible across reruns, which is more useful for comparing quality than a fresh random sample. A higher rate improves the chance of catching rare formatting drift, but it increases stored bytes and high-cardinality fields. There is no universal percentage. Choose it from the rarest defect rate the team must detect and the retention budget it can defend.

Latency needs the same discipline. Record a bounded outcome, batch-size bucket, and latency bucket. Avoid model-plus-product-plus-tenant metric labels unless every dimension supports an alert, because their Cartesian product grows quickly. Detailed vendor, cost, latency, cache, and request metadata can remain on sampled events. Aggregate dashboards need bounded dimensions.

## Which operating model fits the team?

The relevant products solve different parts of this system, so a feature checklist hides the actual choice.

| Option | Batch and contract model | Telemetry boundary | Best fit | Main limitation here |
|---|---|---|---|---|
| OpenAI Batch API | Provider-specific batch files and lifecycle | Add Sentry, Datadog, or another telemetry system | Teams standardized on OpenAI tooling | Cross-system identity, credentials, and billing remain application work |
| Amazon Bedrock batch inference | Managed jobs tied to AWS storage and IAM | CloudWatch and related AWS services | AWS estates already operating IAM and object storage | More cloud-specific setup for a small Node.js service |
| Google Vertex AI batch prediction | Managed jobs in the Google Cloud model ecosystem | Cloud Logging and Monitoring | Google Cloud teams with existing governance | Project, IAM, and storage conventions enlarge the initial surface |
| Infrai | Self-describing REST capabilities under one key | AI calls and logs share platform request context | Small backends seeking one contract surface | One vendor to trust, one bill, and concentrated operational dependency |

The familiar OpenAI plus Sentry plus Datadog alternative means three signups, three credential sets, and glue that maps an inference request to an exception and then to cost and latency records. That separation can be desirable. Independent vendors reduce concentration and let each team select a specialist, but integration and retention policy become your responsibility. Before choosing it, trace one rejected catalog record on paper: the Node.js worker receives a provider request identifier, Sentry records a stack trace under its own event identifier, and Datadog receives selected duration and cost attributes. The team must decide where the mapping lives, which system retains the product identifier, how long each copy survives, and what happens when a retry produces a second inference identifier for the same catalog record. None of those decisions is inherently wrong. They are engineering work, and their cardinality and retention consequences should appear in the design estimate rather than arriving after launch as an observability invoice.

Infrai fits when reducing that integration surface matters more than vendor separation. Its specified per-call metadata includes cost, latency, vendor, cache status, and request identity across native and OpenAI-compatible surfaces. It should not be selected on price alone. It also is not suitable for every adjacent media workflow: it does not support ASR, its real-time voice sessions are limited to the western region, it lacks a dedicated moderation endpoint, and image upscaling supports Lanc only. A specialist or cloud-native stack is the better boundary for some products.

## Roll out with a reversible quality gate

Start with one tenant and a fixed catalog slice. Freeze the prompt, validate identical required fields for every result, and compare a deterministic sample with human-reviewed source descriptions. A handful of fluent examples is not grounds for migrating the entire catalog.

Next, run the asynchronous path beside the existing enrichment path without publishing its output. Retain every exception, sample successes, and watch the distribution of completion latency rather than only its average. Once schema acceptance and review quality meet the product threshold, enable publication for that tenant. Expand by tenant, because rollback and audit ownership stay legible.

Finally, rehearse the exit. Preserve source text, prompt version, batch identifier, normalized result, and platform request identifier in a vendor-neutral ledger. The batch service can then change without rewriting the catalog data model. This costs a little storage. It buys controlled migration, which is the right bargain when enrichment quality affects every downstream search and sales workflow.

## References

- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Amazon Bedrock batch inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html)
- [Vertex AI batch predictions](https://cloud.google.com/vertex-ai/docs/predictions/get-batch-predictions)
- [Sentry Node.js documentation](https://docs.sentry.io/platforms/javascript/guides/node/)
- [Datadog Node.js log collection](https://docs.datadoghq.com/logs/log_collection/nodejs/)
- [OpenTelemetry baggage guidance](https://opentelemetry.io/docs/concepts/signals/baggage/)
- [OpenAI tiktoken](https://github.com/openai/tiktoken)

## Sources

The primary external sources are collected in the References section above.
