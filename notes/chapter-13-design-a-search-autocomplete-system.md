# Chapter 13: Design a Search Autocomplete System

Also called "design top-k" or "design top-k most searched queries" — the feature behind Google/Amazon search suggestions as you type.

## Requirements

Matches only at the **beginning** of a query (prefix matching, not substring); return **5** suggestions ranked by historical popularity; no spell check/autocorrect; English only (lowercase, no special characters) for now; **10 million DAU**.

**Non-functional**: fast (Facebook's own autocomplete targets **under 100ms**, or it visibly stutters), relevant, sorted by popularity, scalable, highly available.

## Back-of-envelope estimation

```
Query string size: 4 words × 5 chars avg = 20 bytes/query

Requests per completed query: ~20 (one HTTP request per keystroke —
  typing "dinner" fires 6: q=d, q=di, q=din, q=dinn, q=dinne, q=dinner)

QPS = 10,000,000 users × 10 queries/day × 20 chars ÷ 24h ÷ 3600s ≈ 24,000
Peak QPS = 24,000 × 2 = ~48,000

New data/day = 10M × 10 queries × 20 bytes × 20% new ≈ 0.4 GB/day
```

> The "20 requests per query" figure is a self-inflicted multiplier from the UX itself — every keystroke fires a request. That's exactly why the sub-100ms latency requirement is non-negotiable: a user isn't submitting one request and waiting, they're typing continuously, and any perceptible lag on any keystroke breaks the experience.

## High-level design — the naive version

**Data gathering service**: maintains a frequency table `{query_string → frequency}`, updated in real time as users search.

**Query service**: given a prefix like "tw", run `SELECT query FROM table WHERE query LIKE 'tw%' ORDER BY frequency DESC LIMIT 5`.

This is a deliberate strawman (the same rhetorical move as Chapter 6/10's naive first designs) with two separate weaknesses, each fixed later: a `LIKE`-based SQL scan doesn't stay fast as the table grows (→ trie), and real-time per-keystroke writes don't scale to real volume (→ batch pipeline).

## The trie data structure, and the naive lookup algorithm

**Trie** ("retrieval") — a tree storing strings compactly. Root = empty string; each node holds a character and up to 26 children; a node can mark a complete word. Frequency gets stored directly on terminal nodes to support ranking.

**Naive top-k algorithm** (`p` = prefix length, `c` = children under a node):
```
1. Walk to the prefix node — O(p)
2. Traverse the subtree, collect valid children — O(c)
3. Sort by frequency, take top k — O(c log c)
```
Worked example: `k=2`, prefix "tr" → find node "tr" → subtree yields `[tree:10],[true:35],[try:29]` → sort, top 2 = `[true:35],[try:29]`.

**Why this is too slow**: the same prefix gets queried repeatedly by many users while the underlying data barely changes between queries — re-walking and re-sorting on every request recomputes an answer that was already computed moments ago. That's exactly the shape of problem where caching the result once beats optimizing the computation.

## Two optimizations — down to O(1)

1. **Limit the max prefix length** — users rarely type long queries, so bound `p` by a small constant (~50). "Find the prefix" drops to `O(1)`. Grounded in real usage, the same move as Chapter 9's "nobody scrolls back thousands of posts."
2. **Cache top-k queries at every node** — pre-compute and store the top-5 directly on each node at build time. Querying becomes: walk to the node (`O(1)`), read the pre-sorted list (`O(1)`). Space traded for time — worth it given the hard latency requirement. Example: node "be" stores `[best:35, bet:29, bee:20, be:15, beer:10]` directly.

**Combined**: `O(1)` end to end.

> **What this actually is**: pre-compute the expensive answer at build time so query time is a trivial lookup — the same move as Chapter 11's fanout-on-write and Chapter 6's write path, just showing up at the **algorithmic/data-structure level** (baked into a tree node) rather than the systems/infrastructure level (a cache, a fanout job).

## Data gathering service, redesigned as a batch pipeline

**Why not real-time**: updating on every query would contend with the query service and defeat the point of pre-computed top-k caching; and top suggestions barely change once built, so real-time updates add little value most of the time.

```
Analytics Logs → Aggregators → Aggregated Data → Workers → Trie DB / Trie Cache
```
- **Analytics Logs** — raw, append-only, unindexed.
- **Aggregators** — the interval is an explicit design decision (Twitter-like: short intervals for freshness; Google-like: weekly is fine) — Chapter 3's "ask, don't assume" as a concrete parameter.
- **Aggregated Data** — a growing time-series table: `(query, week_start, weekly_frequency)`.
- **Workers** — build a fresh trie on schedule from the aggregated history.
- **Trie Cache / Trie DB** — in-memory cache (weekly snapshot) and persistent storage.

> **A subtlety worth being precise about (from a Q&A on this)**: the aggregated data layer accumulates one row per query *per week*, forever — it's not overwritten each week. When workers rebuild, they **sum across a query's full history of weekly rows** (e.g. x: 40 total + 2 this week = 42) to get its current total, then bake that into the new trie. "Replace the old trie" refers to swapping the finished tree object atomically (avoiding a half-updated structure or downtime) — it does **not** mean discarding accumulated counts and starting over from zero each week. One open design question the book doesn't resolve: sum over *all time* (risk: stale historical popularity resists new trends) vs. a rolling/decaying window — this tension is exactly why the wrap-up flags real-time trending as genuinely hard for this design.
>
> **Distinct from the buffer-then-flush pattern** (Ch1/6/9): that pattern *incrementally* accumulates small writes into an ever-growing durable store. This pipeline instead periodically **rebuilds the whole trie from scratch** and **replaces** it wholesale — related instinct ("don't do the expensive thing on every write"), different mechanism (regenerate-and-swap vs. accumulate-in-place).

## Trie storage — two options

- **Document store** — serialize the whole trie into one blob per snapshot (MongoDB fits well). Matches how the Trie Cache wants to work ("load one blob, deserialize into memory"). Downside: can't fetch part of the trie without loading and deserializing everything.
- **Key-value store** — prefix → key, node data → value (Chapter 6's KV store). Matches the actual query pattern much better — a query only ever needs one prefix's data, never the whole tree. Also naturally **shardable via consistent hashing (Ch5)**, since the trie becomes many independent key-value pairs rather than one monolithic object — setting up the sharding topic below.

## Query service deep dive

```
1. Search query → load balancer
2. Load balancer → API servers
3. API servers fetch from Trie Cache, construct suggestions
4. Cache miss → replenish from Trie DB for future requests
```
Same cache-aside, read-through pattern seen repeatedly (Ch1's cache tier, Ch6's read path, Ch8's redirect flow).

**Three optimizations, each solving a distinct problem**:
- **AJAX requests** — each keystroke's request goes out asynchronously, no page reload. Given ~20 requests per completed query, this isn't optional polish — it's what makes search-as-you-type physically viable in a browser at all.
- **Browser caching** — Google's real response header: `Cache-Control: private, max-age=3600`. `private` means the response is for one specific user only and must never be cached by a shared intermediary (a CDN, a proxy) — appropriate since suggestions can be personalized.
- **Data sampling** — log only 1 in N requests, given ~48K peak QPS. A genuinely different tradeoff flavor than most of this book: trading away *completeness of ground-truth data* for cost, rather than trading space or latency. Safe here specifically because ranking only cares about **relative** popularity between queries, not exact absolute counts — uniform sampling preserves relative order even as absolute numbers shrink.

## Trie operations

**Create** — built by workers from aggregated data (topic on the pipeline, above).

**Update — two options, and a hidden cost topic 4 didn't mention**:
1. Rebuild the whole trie weekly, replace wholesale — the default.
2. Update one node in place — avoided, because it's slow. Updating one node's frequency forces updating **every ancestor up to the root**, since ancestors cache their descendants' top-k. A single leaf change can cascade the full depth of the tree.

> The earlier "space for time" framing (topic 4) turns out to be incomplete — the real tradeoff was **space *and* update cost, for read speed**. Caching top-k everywhere makes reads trivial but makes point updates expensive proportional to tree depth. In-place updates are only "acceptable" per the book if the trie is small enough that the cascade stays cheap.

**Delete — a two-layer moderation design**: a **filter layer** sits in front of the Trie Cache, removing hateful/violent/dangerous suggestions at serving time immediately. The underlying bad data is *also* removed from the database asynchronously, so the next scheduled rebuild naturally excludes it from source data too. Same combination-move as Chapter 10's "at-least-once + async dedup": fast reactive fix at the serving layer, paired with a slower eventual correction at the source-of-truth layer.

## Scaling the storage — sharding, and why consistent hashing isn't the fix here

**Naive approach**: shard by first letter (`a-m`/`n-z` for 2 servers; up to 26 servers, one per letter; beyond that, shard on the second character too).

**The imbalance problem — same symptom as Chapter 5, genuinely different cause**: far more English words start with 'c' than 'x'. Consistent hashing's uneven-partition problem comes from *randomness* in hash placement, fixed with more virtual nodes because the underlying data is assumed uniform. Here, the imbalance is **not random** — the keyspace itself (English vocabulary) is permanently, predictably lumpy. More virtual nodes wouldn't help at all, because the problem was never about server placement.

**The fix — a shard map manager**: analyze historical query volume per letter, group letters into shards by roughly **equal actual load**, not equal alphabet range (e.g., if 's' alone carries as much traffic as 'u' through 'z' combined, make two shards: `s` and `u-z`). Maintained as an explicit lookup table, recomputed periodically as patterns shift.

> **A genuinely distinct sharding tool worth keeping alongside consistent hashing**: when a key's distribution is *known and predictably skewed* (not effectively random), a data-driven lookup-table shard map can outperform a hash-based approach, because it directly accounts for measured real skew instead of statistically spreading load and hoping the distribution cooperates.

## Wrap-up — follow-up questions

- **Multi-language support** — store Unicode characters in trie nodes instead of assuming ASCII/lowercase-only.
- **Different top queries per country** — build separate tries per country/region; store them in CDNs for faster response time.
- **Real-time trending queries** — the original design breaks down: offline workers aren't scheduled to rebuild yet (weekly cadence), and even if triggered, a full rebuild takes too long. Genuinely hard, and only briefly sketched: reduce the working dataset via sharding, weight recent queries more heavily in the ranking model, and recognize that trending data often arrives as continuous **streams**, requiring different tooling entirely (Hadoop MapReduce, Spark Streaming, Storm, Kafka) — each requiring its own domain expertise, out of scope here.

## Reference materials
- The Life of a Typeahead Query: https://www.facebook.com/notes/facebook-engineering/the-life-of-a-typeahead-query/389105248919/
- How We Built Prefixy: A Scalable Prefix Search Service: https://medium.com/@prefixyteam/how-we-built-prefixy-a-scalable-prefix-search-service-for-powering-autocomplete-c20f98e2eff1
- Prefix Hash Tree — An Indexing Data Structure over Distributed Hash Tables: https://people.eecs.berkeley.edu/~sylvia/papers/pht.pdf
- MongoDB: https://en.wikipedia.org/wiki/MongoDB
- Unicode FAQ: https://www.unicode.org/faq/basic_q.html
- Apache Hadoop: https://hadoop.apache.org/
- Spark Streaming: https://spark.apache.org/streaming/
- Apache Storm: https://storm.apache.org/
- Apache Kafka: https://kafka.apache.org/documentation/

## My open questions / follow-ups
- *(add doubts here as they come up)*
