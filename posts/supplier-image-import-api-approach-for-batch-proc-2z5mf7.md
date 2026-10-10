# Supplier Image Import API Approach for Batch Processing at Scale

**Short answer:** remove backgrounds during upload, after exact-byte deduplication and normalization, when supplier photos feed a logistics catalogue in bulk. Store the original, normalized source, and derived cutout under separate immutable keys. On-demand processing remains a fallback for cold images, not the default path. This moves failures ahead of publication and makes a batch reproducible without turning each catalogue request into an image-processing request.

The decision is really about where uncertainty is allowed. Supplier archives contain repeated files, inconsistent formats, and records that no longer match their images. A useful pipeline gives each uncertainty a boundary and records counts at that boundary. It does not attach a supplier, SKU, filename, or digest to every metric label; those values belong in a manifest or sampled diagnostic event, where cardinality can be controlled.

## How should an API import supplier images at scale?

The intake contract has four invariants. Every accepted catalogue row resolves to one captured source object. The byte digest is computed over that captured object, not over a URL that may later yield other bytes. Normalization has a policy version. Publication points to a completed derivative only after the manifest records its lineage.

For a Nodejs coordinator, the practical approach is still language-independent: capture, dedupe exact bytes, normalise under a versioned policy, then batch the expensive background-removal work. The runtime does not change the data invariants.

A byte digest answers a narrow question: have these exact bytes entered this pipeline? It does not prove that differently encoded files depict the same photograph. Exact-byte deduplication can safely suppress repeated work; perceptual matching is a separate review policy with false-positive consequences, especially when two package variants differ only in a small label.

Failures follow the data. Fetch errors remain in capture. Rejected media remains in inspection. Normalization failures never reach background removal, and a failed cutout never replaces the published asset. A batch can resume from its manifest instead of replaying successful downloads and transformations.

Count every transition. Keep the metric dimensions bounded: `stage`, `outcome`, `format_family`, and a controlled error class are useful candidates. A raw URL or SKU is not. One supplier with 500,000 rows should change counter values, not create 500,000 time series. This design has a real limitation: upload-time work delays batch completion and retains more intermediate data than a purely on-demand path. It is not suitable when nearly all inventory is cold, import completion has a strict short deadline, or originals cannot be retained under the governing data policy. In those cases, the on-demand design in the rejected-option section is the better fit.

That is the trade-off.

## The architecture decision

| Decision axis | Process at upload | Process on demand |
|---|---|---|
| Publication | Publish completed derivatives | First access needs generation or a pending state |
| Duplicate work | Manifest suppresses exact repeats early | Shared result keys and coordination are required |
| Failure location | Import workflow owns retry and quarantine | Read workflow inherits processing failures |
| Capacity shape | Queue absorbs supplier bursts | Reader traffic determines concurrency |
| Best fit | Active inventory with predictable presentation | Large cold archive with sparse access |

Upload-time processing wins here because catalogue import already has a natural completeness boundary. The batch is not done merely because files were fetched. It is done when accepted rows have publishable derivatives or explicit terminal reasons. The storefront reads an asset instead of initiating media work.

This consumes storage for intermediates and derivatives, so retention must be intentional. Let `N` be accepted unique sources, `So` the mean original bytes, `Sn` the mean normalized bytes, and `Sd` the mean derivative bytes. One retained generation occupies approximately `N x (So + Sn + Sd)` bytes before replication and metadata overhead. Keeping three obsolete normalized generations multiplies the relevant term. That should be a policy decision, never an accident caused by an unrecorded version.

Keep originals according to recovery requirements. Keep normalized sources long enough to reproduce or audit the active derivative. Expire abandoned staging objects quickly when operational and compliance rules permit. Measure object counts and bytes by lifecycle class rather than inferring them from request volume.

## Critical path and idempotency

A coordinator creates one durable manifest before fan-out. Each row records a source locator, capture status, byte digest after download, normalization version, derivative version, and terminal outcome. Workers claim rows through a bounded ownership mechanism; retries reuse an idempotency key derived from immutable inputs and policy versions.

These calls illustrate a deliberately small generic boundary. The documentation hostname is non-operational: submit a manifest, inspect aggregate state, then retrieve the completed manifest.

```bash
curl --request POST 'https://media.example.invalid/v1/image/batch/submit' \
  --header 'Content-Type: application/json' \
  --header 'Idempotency-Key: shipment-catalogue-1842-policy-7' \
  --data '{"manifest_uri":"s3://catalogue-intake/manifests/1842.json","normalization_policy":"norm-7","derivative_policy":"cutout-3"}'

curl --request GET 'https://media.example.invalid/v1/image/batch/status/shipment-catalogue-1842' \
  --header 'Accept: application/json'
```

Do not derive idempotency from the supplier URL alone. The same locator can lead to another capture, while identical captured bytes processed under a new normalization or cutout policy represent legitimate new work. The input digest plus policy versions describes the operation.

Batching should bound concurrency, not erase item outcomes. One malformed file must not force a replay of 9,999 successful files. Conversely, a lone `batch_failed` event hides whether the cause was one bad object or a systemic decoder problem. Preserve row outcomes in the manifest, then derive low-cardinality counters from them.

## Observability without runaway cardinality

Start with questions, then decide what telemetry earns retention. Operations needs throughput at each stage, queue age, terminal outcome counts, duration distributions, bytes read and written, and unique-source counts after exact deduplication. These signals support capacity and correctness decisions without converting identifiers into dimensions.

Logs serve another purpose. Emit a structured event when a row changes state, but retain routine successes for less time than failures when audit rules allow it. Sample successful diagnostics deterministically by stable row identifier so repeated investigations see the same cohort. Do not sample terminal failures until their bounded classification and manifest persistence are confirmed.

Trace sampling needs the same asymmetry. A small baseline sample can describe healthy latency; slow or failed traces deserve a higher retention probability. The trace carries a correlation token that resolves through controlled storage to the manifest row. Copying full supplier URLs and descriptions into every span increases stored bytes, cardinality, and exposure without improving aggregate signals.

No SKU labels.

Cardinality needs a budget before launch. If `stage` has 5 values, `outcome` has 4, `format_family` has 6, and `error_class` has 8, their full cross-product is 960 combinations before environment or region. The labels will not always combine freely, but multiplication is the warning. An unbounded SKU changes the scale from hundreds of possible series to one per item.

Retention math belongs beside the event schema. At 500 bytes per event, 2 million row transitions produce about 1 GB before indexing, replication, and other storage overhead. This is arithmetic, not a benchmark. Measure encoded event size in the selected system, multiply it by observed volume and retention, then include the system's actual overhead.

Numbers first.

## Rejected option and its valid use case

The rejected default is background removal on the catalogue read path. It couples presentation latency and availability to derivative generation, while allowing a supplier defect to appear only when someone requests the item. Load also follows attention: a newly popular SKU can need processing while the read path is busiest.

Cold data changes the answer.

On-demand work is valid for a deep archive where few images will ever be displayed. Use an immutable source key and policy version to address the derivative, coordinate concurrent misses, return a defined pending state, and keep generation outside the request's critical execution window. Promotion into active inventory can precompute the cutout. The invariant remains: readers never observe a partially written asset.

A hybrid policy is defensible. Process active inventory at upload, defer explicitly classified cold objects, and record the tier in the manifest. Revisit the threshold using measured access counts, derivative bytes, queue age, and miss behavior rather than a changing API price.

## Decision record

Adopt capture, exact-byte deduplication, versioned normalization, and background removal as an upload-time workflow for active logistics catalogue imports. Publish immutable derivative references only after item completion. Reserve on-demand generation for classified cold inventory, with shared coordination and a visible pending contract.

The acceptance test is concrete: replaying the same captured bytes under the same policy versions creates no extra derivative; a policy change creates a new addressable derivative; one rejected image does not replay completed rows; aggregate telemetry remains bounded as the catalogue grows. This protects the read path and keeps retained observability data tied to useful questions.

## References

- MDN Web Docs, "Image file type and format guide": https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
