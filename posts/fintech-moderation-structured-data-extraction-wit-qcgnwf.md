# Fintech Moderation: Structured Data Extraction with LLM 429 Limits and US/EU Queue Design

Short answer: for fintech moderation reports, treat an LLM 429 rate limit as an admission-control event, not as a malformed structured data extraction. Put JSON extraction behind bounded regional queues, keep retry state with the job, and make the provider call a replaceable Python adapter. Batch APIs are useful for replayable review backlogs; they are the wrong lane for a report waiting on a human decision. This is a scheduling problem before it is a model-selection problem.

Backpressure first.

The concrete flow is small, but each handoff deserves a name. A report arrives with its source ID, gets assigned a residency region and schema version, and enters the matching queue. Admission checks whether the lane has room and records a prompt-size estimate before a worker claims the item. The worker calls an LLM through an adapter, validates the returned JSON, and writes a result keyed by the original report ID. A valid classification becomes ready for human review; an invalid object becomes an inspection item with its own reason; a capacity response returns to the same regional queue with an attempt count and a deadline. A batch import uses the same validator and result store, but its progress is reconciled from chunk records rather than returned to a waiting HTTP caller. That distinction keeps a notebook-to-prod path intact while making provider portability a property of the application instead of a promise in a comparison table. It also gives an eval harness a stable place to compare schema-pass rate, queue age, retry volume, and prompt cost across adapters.

## How can a fintech moderation queue keep JSON extraction moving when US and EU rate limits diverge?

Start with admission. A worker should not accept unlimited work merely because the process can create more tasks. Give the US and EU lanes separate concurrency limits, queue depth, and observability labels. A retry stays in its original lane. That detail matters: silently moving a report to another region during backoff can turn a capacity decision into a data-governance decision.

The queue item should carry an input ID, region, schema version, prompt-size estimate, and attempt count. The ID makes the result write idempotent. The schema version lets an old job finish against the contract it was admitted under. The estimate is not a billing oracle; it is a scheduling signal that prevents a long report from looking equivalent to a short one.

Backoff needs a deadline as well as a delay. When the provider communicates a retry interval, use it as the lower bound. Add jitter so workers do not wake in the same millisecond, and stop after a bounded number of attempts or a job deadline. A 429 gets this treatment. Invalid input, authentication failure, and schema rejection go to their own paths; retrying those conditions only hides the useful signal.

Here is the smallest adapter I would put around that policy. The provider-specific implementation is intentionally outside the worker. The demo provider makes the control flow executable without inventing a commercial endpoint; replace it with an adapter that returns a JSON string and raises `RateLimited` for a capacity response.

```python
import asyncio
import json
import random
from dataclasses import dataclass
from typing import Awaitable, Callable


class RateLimited(Exception):
    def __init__(self, retry_after: float | None = None) -> None:
        self.retry_after = retry_after


@dataclass(frozen=True)
class Report:
    input_id: str
    region: str
    text: str


Provider = Callable[[Report], Awaitable[str]]


async def demo_provider(report: Report) -> str:
    # A real adapter would call one provider and return its JSON content.
    await asyncio.sleep(0)
    return json.dumps({"input_id": report.input_id, "decision": "review"})


def validate_json(content: str) -> dict[str, object]:
    value = json.loads(content)
    if not isinstance(value, dict):
        raise ValueError("extraction must be a JSON object")
    required = {"input_id", "decision"}
    if not required.issubset(value):
        raise ValueError("extraction is missing a required field")
    return value


async def extract(
    report: Report,
    provider: Provider,
    max_attempts: int = 5,
    sleep: Callable[[float], Awaitable[None]] = asyncio.sleep,
) -> dict[str, object]:
    for attempt in range(max_attempts):
        try:
            return validate_json(await provider(report))
        except RateLimited as error:
            if attempt == max_attempts - 1:
                raise
            exponential = min(30.0, 2**attempt)
            delay = max(error.retry_after or 0.0, exponential + random.random())
            await sleep(delay)
    raise RuntimeError("retry loop exhausted")


async def main() -> None:
    reports = [
        Report("case-001", "US", "A transfer was reported as suspicious."),
        Report("case-002", "EU", "A card listing was reported as misleading."),
    ]
    results = await asyncio.gather(
        *(extract(report, demo_provider) for report in reports)
    )
    print(json.dumps(results, indent=2))


if __name__ == "__main__":
    asyncio.run(main())
```

The worker count is only one part of the limit. The queue's maximum size is the earlier brake, and a per-region semaphore is the other brake when several kinds of work share a process. In a service, persist the job before dispatch, acknowledge it only after an idempotent result write, and record the final status when the retry budget is spent. I've seen the most confusing dashboards come from combining transport retries and validation failures into one counter. Keep them separate.

## What changes when the backlog becomes a batch API job?

Latency is the dividing line. An analyst waiting for a classification belongs in the online lane. A closed month's moderation reports can be split into restartable chunks and processed as a batch, provided the batch interface exposes enough status to reconcile submitted, completed, and failed items. The application still needs the same schema validator; a batch response is not trustworthy merely because it arrived later.

Do not use a batch lane as a disguise for unbounded work. Select a chunk size from measured prompt size, expected retry allowance, and the maximum time an operator is willing to wait for a replay. Store the input ID beside every submitted item. If a process stops after 83 of 100 records, the next run should identify the missing 17 rather than submit all 100 again.

The decision should come from an eval harness, not queue intuition. Use a fixed sample of reports containing missing fields, ambiguous language, long descriptions, and deliberately malformed source text. Measure schema-pass rate, human-review agreement, queue age, 429 frequency, and prompt tokens independently. A low 429 count with a poor schema-pass rate is not a capacity win. It is a bad extraction.

Batch work also changes failure handling. A synchronous request can return a useful status to its caller; a batch needs a reconciliation record and an operator-facing retry policy. Preserve the original region and schema version in that record. If a provider cannot express the controls the job requires, the correct response is to choose an adapter or execution path that can, not to scatter provider-specific assumptions through the queue.

## Which provider contract keeps US/EU extraction portable?

Portability is an interface decision. The application should own the report ID, region, schema version, validation result, retry budget, and idempotency key. The adapter should own authentication, request serialization, response normalization, and provider-specific status mapping. This boundary makes a provider change visible in one place and keeps the eval corpus reusable.

| Contract choice | Useful when | Trade-off |
|---|---|---|
| Common JSON request and response shape | Several providers must serve the same moderation job | Provider-specific controls may need an explicit escape hatch |
| Provider-native client | One backend's features are central to the workflow | Switching backends can touch queue and validation code |
| Self-hosted inference adapter | Data handling and capacity are owned by the team | Serving, scaling, batching, and model updates become team responsibilities |
| Batch-first interface | The corpus is large, restartable, and not latency-sensitive | Human-facing decisions need a separate online path |

The catch is that a common contract is not the same as identical behavior. JSON mode, token accounting, retry headers, regional availability, and batch semantics can differ. Keep those differences in adapter tests and label each eval result with the adapter and model configuration that produced it. Your mileage may vary by account-level capacity and deployment region, so treat a concurrency number as a starting hypothesis, not a permanent setting.

This is also where the Python API should stay boring. One function accepts a report and returns a validated object; it should not know which SDK created the response. The notebook can use the demo adapter, a staging adapter can replay recorded fixtures, and production can select a regional implementation through configuration. That shape protects the job from a rewrite when the preferred backend changes.

## What should be checked before shipping a moderation extractor?

Replay the eval corpus through both regional lanes and the batch reconciler. Verify that a retry never changes region, that a duplicate input ID cannot create a duplicate decision, and that a validation failure does not consume the rate-limit retry budget. Force a synthetic capacity response and confirm that backoff spreads work instead of producing a second spike.

Then inspect the boring numbers: oldest queue item, per-region concurrency, attempts per input, schema failures, token estimates, and completed batch items. Three words matter: measure the boundary.

If the workload is small and interactive, use the online path. If it is a large, restartable corpus, use batch processing. Stay with the provider-native contract when its controls are part of the compliance requirement; choose a common adapter when changing providers is the larger operational risk. The durable choice is the one that keeps the moderation decision auditable after the next limit change.

## References

- https://docs.cohere.com/docs/rerank-overview
- https://github.com/openai/whisper
