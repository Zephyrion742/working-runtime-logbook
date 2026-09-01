# Implementing Relevance Tuning in a Retrieval Architecture for Changing Product Pages

Use a two-stage design: a cheap change detector that decides which chunks are stale, and a relevance tuning loop that only ever runs against a frozen judgment set. That ordering is the architecture decision; the rest is implementation detail. The constraint that forces it has nothing to do with embedding quality. In a retrieval system whose corpus is a set of watched web pages, the documents move underneath the index, so every relevance number you collect measures two variables at once — ranking behaviour and staleness — and a system whose ground truth shifts while you measure it cannot be tuned in any honest sense.

The system I will hold fixed throughout is an e-commerce catalog watcher: roughly 12,000 online product pages, polled four times a day, feeding an assistant that answers questions about price, stock, shipping windows and return policy. Diff first, alert second, re-embed third.

## The invariants that make relevance measurable

Three properties have to hold before any tuning number means anything, and each of them is an ingestion decision rather than a ranking decision.

Chunk identity must be content-addressed and stable across crawls. A chunk id of `sku:block` — `9F2K:shipping`, `9F2K:returns` — lets a re-crawl overwrite a row in place. Skip this and the index quietly accumulates the March and the June version of the same shipping paragraph, both of them plausible, both of them retrievable, and a reranker will happily return the pair. Then a human reads the older one.

Every indexed chunk carries `observed_at` and the ETag it came from. Retrieval that cannot state when it saw a price is not auditable, and in a catalog the difference between a 20-minute-old and a six-day-old shipping promise is the difference between a correct answer and a chargeback.

The third invariant is the one teams skip: the evaluation corpus is pinned to a snapshot, never to the live site.

That last point is where the observability bill starts to matter, so here's the retention math I'd defend in review. Twelve thousand pages at nine blocks each is about 108,000 hashes per crawl; at 32 bytes a hash that's 3.5 MB per crawl, 14 MB a day, roughly 1.2 GB if you keep ninety days of diff history. Keeping the rendered HTML for the same window is a different order of magnitude entirely — at 180 KB a page you are storing 2.1 GB per crawl, about 8.6 GB a day, close to 780 GB for those ninety days. So keep hashes long and bodies short: ninety days of block hashes gives you the full change timeline for relevance attribution, and seven days of raw HTML (around 60 GB) is enough to debug any diff a human actually asks about. The same discipline applies to metrics. Do not put the SKU on a metric label; 12,000 products times four series is 48,000 active series for a dashboard that nobody reads per-product anyway, and a single `block_type` label with nine values answers the question you actually have.

## How should relevance tuning change when the retrieval corpus keeps moving?

It stops being one activity and becomes two loops running at different frequencies.

The ranking loop runs against the pinned snapshot with the index held constant. Sample the query log — I use 200 queries stratified across the intents the assistant actually serves, judge the top ten results for each, and track nDCG@10 across changes to chunk size, hybrid weighting and reranking depth. That's 2,000 judgments to build the set once. Re-judging all of them weekly isn't affordable for a team of three, and it isn't necessary: only the query-document pairs whose chunk hash moved need a fresh label, which at a typical 3% block change rate per crawl is a few dozen pairs, not two thousand. I'm not sure there's a principled argument for 200 rather than 500 queries beyond judging budget; what I can defend is that the number stays fixed while the knobs move.

The freshness loop measures something the ranking loop structurally cannot see: the p95 age of the chunks that appeared in served answers, bucketed by block type. Price blocks and stock blocks get a tight objective — an hour, say. Return-policy blocks can drift for a week without anyone being harmed. Tuning those two objectives separately is the whole reason the block-level chunking exists.

Conflating the loops produces the failure mode I see most often in incident reviews for this kind of system: an answer-quality regression gets attributed to a model or a chunk-size change, three days go into ranking experiments, and the actual cause was a supplier who reworded a shipping table on 400 pages at once.

## Chunk boundaries that survive an edit

Chunking here is not primarily a relevance choice. It is a blast-radius choice, and relevance follows from it.

| Chunking strategy | Blast radius of a one-line edit | Relevance behaviour | Operational cost |
|---|---|---|---|
| Whole page, one vector | Entire page re-embedded | Specific questions drown in nav and boilerplate | Lowest to build, worst per-edit cost |
| Fixed 512-token windows | Every window after the edit point shifts | Decent recall, poor attribution | Cheap, but diffs are meaningless |
| Semantic blocks from DOM structure | One block | Answers map to a named section | Extraction rules need maintenance per template |
| Sentence-level with parent context | One sentence, plus its parent | Highest precision on short factual queries | Largest index; roughly 6x the rows |

Fixed windows are what most tutorials show, and they're the one option that actively fights a change-watching architecture: insert a sentence near the top of a page and every downstream window shifts, so the hash diff reports that the entire document changed. The re-embed cost goes from one block to nine, and worse, the diff signal that was supposed to drive alerting becomes noise.

Semantic blocks derived from the page's own structure — a heading, a specification table, a policy section — are the ones I'd default to for a catalog, with `schema.org/Product` markup used as the extraction anchor wherever a merchant publishes it. The catch is that the extraction rules are template-specific and they rot; each storefront template redesign costs a day of rule maintenance, and there's no way to detect that failure except by asserting on expected block counts per template.

## Watching the page and re-embedding only what moved

The critical path is a conditional fetch, a hash comparison, then embedding work scoped to the delta. Conditional requests are the cheapest part of the entire design and the most frequently omitted.

```bash
ETAG=$(cat state/9F2K.etag 2>/dev/null || echo '"none"')
CODE=$(curl -sS -o page.html -w '%{http_code}' \
  -H "If-None-Match: $ETAG" \
  https://shop.example.com/p/9F2K-espresso-grinder)

test "$CODE" = "304" && exit 0

./extract-blocks page.html --out blocks/
sha256sum blocks/* | sort > blocks.sha256
comm -13 state/9F2K.sha256 blocks.sha256 | awk '{print $2}' > changed.txt
```

A 304 ends the run before a single byte of HTML is parsed or a single embedding is bought. On a catalog that reprices weekly, the majority of polls terminate right there, which is what makes four-times-daily polling defensible in the first place.

Only the files listed in `changed.txt` reach the embedding endpoint, and each upsert carries the provenance the invariants demand:

```bash
while read -r f; do
  BLOCK=$(basename "$f" .txt)

  VEC=$(curl -sS https://embed.internal/embeddings \
    -H "Authorization: Bearer $EMBED_TOKEN" \
    -H 'Content-Type: application/json' \
    -d "$(jq -n --rawfile t "$f" '{model: env.EMBED_MODEL, input: $t}')" \
    | jq -c '.data[0].embedding')

  curl -sS -X POST https://index.internal/collections/catalog/points \
    -H "Authorization: Bearer $INDEX_TOKEN" \
    -H 'Content-Type: application/json' \
    -d "$(jq -n --arg id "9F2K:$BLOCK" --argjson v "$VEC" --arg block "$BLOCK" --arg etag "$ETAG" \
      '{points: [{id: $id, vector: $v, payload: {sku: "9F2K", block: $block, observed_at: (now | todate), source_etag: $etag}}]}')"
done < changed.txt
```

Two implementation notes that cost me more time than the ranking work did. Alert on the semantic diff of the price and policy blocks, never on a byte diff of the page: rotating hero images, CSRF tokens and a nav banner counting down a sale will fire every hour and train the on-call to ignore the channel. And retire chunks in the same transaction that writes the new ones, because a deleted specification row whose vector survives is a fact the assistant will keep asserting long after the merchant stopped publishing it. Postgres with pgvector keeps the row and its vector under one transaction boundary, which makes that retirement trivial; it does not hand you a managed sharded index once the collection outgrows a single node, and that's a real ceiling worth knowing before you pick it. Elasticsearch and OpenSearch expose BM25 parameters directly, so a lexical baseline stays tunable without retraining anything — at the price of running a cluster whose failure modes your team now owns.

Hybrid retrieval matters more here than in most corpora, because catalog queries are full of exact tokens — model numbers, dimensions, `9F2K` — where lexical matching beats dense vectors outright, while policy questions are paraphrase-heavy and go the other way.

## The option I rejected, and where it is the right call

The rejected design is the nightly full re-embed: crawl everything, re-chunk everything, rebuild the collection, swap an alias. It is dramatically simpler, it has no diff ledger, no hash state, no extraction assertions, and I have watched teams ship it in an afternoon.

I rejected it on two grounds. Cost scales with corpus size rather than with change, so 108,000 embeddings a night buys you the same information as roughly 3,200; and a full rebuild erases the change timeline, which is the only artifact that lets you attribute a relevance regression to a specific content edit rather than to your own tuning.

That verdict flips completely when the corpus is slow. Consider the retrieval architecture behind an online course tutor: a syllabus, a set of lecture notes, a problem bank, all of which change a handful of times per term and mostly between terms. Freshness is nearly free there, the entire diff apparatus is dead weight, and the tutor's relevance work belongs somewhere else — query rewriting over a small fixed vocabulary, and making sure a retrieved passage carries its week number and prerequisite chain so the answer can cite where in the course it came from. Stick with the scheduled full rebuild whenever the corpus changes less often than you re-tune the ranker. Adopt the diff-gated pipeline when the reverse is true, which for commerce content it almost always is.

## References

- Lewis et al., Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks — https://arxiv.org/abs/2005.11401
- RFC 9110, HTTP Semantics (conditional requests, ETag, 304) — https://www.rfc-editor.org/rfc/rfc9110.html
- Järvelin and Kekäläinen, Cumulated gain-based evaluation of IR techniques (nDCG) — https://dl.acm.org/doi/10.1145/582415.582418
- Robertson and Zaragoza, The Probabilistic Relevance Framework: BM25 and Beyond — https://dl.acm.org/doi/10.1561/1500000019
- Thakur et al., BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of IR Models — https://arxiv.org/abs/2104.08663
- schema.org Product vocabulary — https://schema.org/Product
