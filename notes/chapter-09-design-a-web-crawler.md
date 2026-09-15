# Chapter 9: Design a Web Crawler

A web crawler (robot/spider) starts from a set of pages and follows links outward to discover more. **Uses beyond search indexing**: web archiving (national libraries preserving the web), web mining (financial firms scraping shareholder reports), web monitoring (detecting copyright/trademark infringement).

**The naive algorithm** looks simple: download pages from a URL list → extract URLs from them → add new URLs back to the list → repeat. The entire rest of this chapter exists because that three-line loop breaks down brutally at real scale.

## Requirements

Clarifying questions establish scope: search engine indexing (not archiving/mining), **1 billion pages/month**, **HTML only** (no PDFs/images), must catch newly-added *and* edited pages, **store crawled HTML for 5 years**, duplicate content should be ignored.

**Four qualities of a good crawler:**
- **Scalability** — the web is huge; needs heavy parallelization.
- **Robustness** — the web is full of traps: malformed HTML, unresponsive servers, malicious links. Must degrade gracefully, never crash.
- **Politeness** — never hammer one host with too many requests in a short window.
- **Extensibility** — adding a new content type shouldn't require redesigning the system.

## Back-of-envelope estimation

```
QPS = 1,000,000,000 / 30 days / 24h / 3600s ≈ 386 → rounds to ~400 pages/sec
Peak QPS = 2 × 400 = 800

Avg page size: 500 KB
Monthly storage = 1 billion × 500 KB = 5×10¹⁴ bytes = 500 TB/month
5-year storage = 500 TB × 12 months × 5 years = 30,000 TB = 30 PB
```
*(Verified — unlike Chapter 8's storage calculation, this one checks out with no double-counting.)*

## High-level design — the components

Eleven components across three stages:

**Getting a URL to download:**
- **Seed URLs** — starting points (chosen by locality or topic; an open-ended design question).
- **URL Frontier** — a FIFO queue of "to be downloaded" URLs (see deep dive below).
- **HTML Downloader** — pulls the next URL and downloads the page.
- **DNS Resolver** — translates a URL to an IP address for the downloader.

**Processing what comes back:**
- **Content Parser** — validates/parses HTML, kept as a *separate* component from the downloader so slow parsing of malformed pages never blocks fetching the next page.
- **Content Seen?** — dedup check via **hash comparison** (not character-by-character — too slow at billions of pages). ~29% of web pages are duplicates.
- **Content Storage** — hybrid: most content on disk (too big for memory), popular content also cached in memory for latency — the same hot/cold split from Chapter 1.

**Discovering the next URLs:**
- **URL Extractor** — pulls links from parsed HTML, converts relative paths to absolute URLs.
- **URL Filter** — excludes certain content types, broken links, blacklisted sites.
- **URL Seen?** — has this URL already been visited/queued? Implemented via bloom filter or hash table (same tool as Chapter 6's read path), prevents redundant work *and* infinite loops.
- **URL Storage** — stores already-visited URLs.

## The 11-step workflow

```
1.  Seed URLs go into the URL Frontier
2.  HTML Downloader pulls a batch of URLs from the Frontier
3.  Downloader resolves IPs via DNS Resolver, then downloads
4.  Content Parser parses the HTML, checks it isn't malformed
5.  Valid content goes to "Content Seen?"
6.  Content Seen? checks storage:
      → already stored (same content, different URL) → discard
      → new content → pass to Link Extractor
7.  Link Extractor pulls links out of the HTML
8.  Extracted links go through the URL Filter
9.  Filtered links go to "URL Seen?"
10. URL Seen? checks: already visited/queued? → do nothing
11. Never seen before? → add it back into the URL Frontier
```
Step 11 feeds directly back into step 1's queue — the loop that makes this a crawler, with every failure mode (duplicates, infinite loops, malformed pages) explicitly guarded by a dedicated component.

## DFS vs. BFS

Model the web as a directed graph — pages are nodes, hyperlinks are edges. **DFS is a poor fit**: it can go arbitrarily deep down one path before backtracking, risking narrow coverage instead of breadth. **BFS is standard**, implemented as a FIFO queue — but naive BFS has two problems:

1. **Impoliteness** — most links on a page point back to the *same host* (Wikipedia's links are mostly to other Wikipedia pages). Naive parallel BFS can fire a burst of simultaneous requests at one server — indistinguishable from a DoS attack.
2. **No prioritization** — standard BFS treats every URL equally. A random forum post mentioning "Apple" gets the same priority as apple.com's homepage.

Both problems point at the same missing piece: a smarter queueing structure between "discovered a URL" and "actually download it" — the **URL Frontier**.

## URL Frontier deep dive

### Politeness (fixes impoliteness), worked through

10,000 URLs discovered from `wikipedia.org`, 200 from `nytimes.com`, 5 from a small blog.
- **Mapping table**: `{wikipedia.org → b1, nytimes.com → b2, smallblog.com → b3}`.
- **Queue router** dumps each host's URLs into its own back queue (`b1, b2, b3, ... bn`).
- **Worker threads**: `W1` only pulls from `b1`, `W2` only from `b2`, etc.

Without this, Wikipedia's 10,000 URLs could occupy most of the crawler's parallel download slots at once. With it, no matter whether `b1` holds 10 or 10 million URLs, exactly **one** worker pulls from it, at a rate capped by that worker's enforced delay — the queue size for a host and that host's actual download rate are structurally decoupled.

> **Connects to a broader lesson**: a computed aggregate capacity number (like our 400-800 QPS) is a *total system budget* — it says nothing on its own about fair allocation among whoever draws from it. Politeness is exactly the mechanism that prevents one host from monopolizing that budget, the same way Chapter 4's rate limiter prevents one client from monopolizing an API's budget. Same underlying problem shape, different key (host vs. client/tenant).

### Priority (fixes no-prioritization), worked through

Three URLs: `apple.com/` (score 9), `en.wikipedia.org/wiki/Apple_Inc.` (score 7), a forum post mentioning "Apple" (score 2) — scored by the **Prioritizer** using PageRank, traffic, update frequency. They land in front queues by band: `f1` (8-10), `f2` (4-7), `f3` (0-3).

The **queue selector** picks among front queues **randomly, but biased toward higher priority** (e.g., ~60%/30%/10% across `f1`/`f2`/`f3`) — not a strict drain-`f1`-first rule, which would let `f3` (and the forum post) **starve forever** if high-priority URLs kept arriving faster than `f1` emptied.

### Front + back queues combined — tracing one URL

`en.wikipedia.org/wiki/Apple_Inc.`:
1. Prioritizer scores it 7 → front queue `f2`.
2. Biased selector eventually picks `f2`, pulls this URL.
3. Queue router looks up `en.wikipedia.org` → routes into back queue `b1` (same queue as every other Wikipedia URL, regardless of priority).
4. Worker `W1` reaches it in FIFO order within `b1`, downloads it.

**The subtlety**: priority determines *how soon a URL enters the politeness system* (jumps ahead at the front-queue stage) — but once inside its host's back queue, it still waits its turn like everything else for that host. The two mechanisms sit at different pipeline stages and don't fight each other.

### Freshness

`nytimes.com/` updates every few minutes → recrawled every 5-10 minutes. A static blog's "About Me" page → recrawled every few months. Two independent, reinforcing signals: **update history** (how often a page actually changes) and **priority** (how much a page matters) — a high-churn but zero-traffic page only gets the first boost, not both.

### Frontier storage, worked through

500 million URLs at ~150 bytes each ≈ 75 GB — bigger than a lot of single-machine RAM, and needs to survive restarts anyway. Meanwhile enqueue traffic can reach several thousand URLs/sec (each crawled page yields dozens of links) — writing each as an individual disk operation means thousands of small random disk I/Os per second, and a disk seek costs ~10ms (Chapter 2's latency table) — disk becomes the bottleneck fast.

**Fix**: an in-memory buffer absorbs enqueue/dequeue traffic; periodically flushed to disk as one large **sequential** write.

### The recurring pattern across this whole book

| Where | Fast/hot layer | Durable/bulk layer |
|---|---|---|
| Ch1 — single-server KV store | In-memory hot data | Disk |
| Ch6 — LSM-tree write path | Memtable + commit log | Immutable SSTable |
| Ch9 — Content Storage | Cache of popular pages | Disk (bulk HTML) |
| Ch9 — URL Frontier storage | Enqueue/dequeue buffer | Disk (bulk URLs) |

**The one underlying trick**: whenever a dataset is too big for memory *and* the access pattern is too frequent/fine-grained for direct disk I/O, buffer the small frequent operations in memory and periodically batch them into large, sequential disk writes. Once recognized, expect to see it again in any new system with "too big for memory, too slow for disk alone" in its requirements.

## HTML Downloader deep dive

### Robots.txt (Robots Exclusion Protocol)

Before crawling any site, check and obey its `robots.txt`. Real example (Amazon):
```
User-agent: Googlebot
Disallow: /creatorhub/*
Disallow: /rss/people/*/reviews
```
**Cache it** rather than re-fetching on every request to that host — refresh periodically.

### Performance optimizations

1. **Distributed crawl** — partition the URL space across servers/threads (Chapter 1's horizontal scaling, applied here) — previews Robustness's use of **consistent hashing** (Chapter 5) for that partitioning.
2. **Cache the DNS Resolver** — DNS lookups are synchronous and slow (10-200ms, per Chapter 2's latency table) and block the requesting thread. Caching domain→IP mappings means only the first request per domain pays that cost; refreshed periodically via cron jobs.
3. **Locality** — place crawl servers (and caches, queues, storage) geographically close to the hosts being crawled — the same principle behind Chapter 1's CDN/multi-data-center reasoning.
4. **Short timeout** — cap how long a worker waits on an unresponsive server, so one dead host doesn't quietly stall a thread and reduce effective throughput.

## Robustness

- **Consistent hashing** (Chapter 5) — distributes load among downloaders; adding/removing a downloader server only reshuffles a small fraction of the URL space.
- **Save crawl state and data** — a disrupted crawl can restart from saved state instead of from scratch.
- **Exception handling** — errors are common at this scale; the crawler must handle them without crashing.
- **Data validation** — guards against corrupt/malformed data propagating through the system.

## Extensibility

New content types get added by **plugging in new modules**, not redesigning the system — e.g., a PNG Downloader module for images, or a Web Monitor module for copyright/trademark tracking. This is the practical payoff of keeping components (downloader, parser, extractor) cleanly separated in the high-level design.

## Detecting problematic content

1. **Redundant content** — ~30% of pages are duplicates; hashes/checksums detect this (same mechanism as "Content Seen?" above).
2. **Spider traps** — a page structure that causes infinite crawling, e.g. `www.example.com/foo/bar/foo/bar/foo/bar/...`. Mitigated with a max URL length, but no fully general automated solution exists — sites with an unusually large discovered-page count are the tell, and often need manual review/exclusion or custom filters.
3. **Data noise** — low-value content (ads, code snippets, spam URLs) excluded where possible.

## Wrap-up talking points

- **Server-side (dynamic) rendering** — many sites generate links via JavaScript/AJAX; downloading raw HTML alone misses those links, so rendering the page first (like a browser would) is needed to see the full link set.
- **Filter out unwanted pages** — an anti-spam component to exclude low-quality/spam pages given finite crawl resources.
- **Database replication and sharding** (Ch1, 5, 6) — for data-layer availability, scalability, reliability.
- **Horizontal scaling** — hundreds/thousands of servers for large-scale crawling; the key requirement is keeping them stateless.
- **Availability, consistency, reliability** (Ch1) — the general principles apply here too.
- **Analytics** — collecting/analyzing crawl data to keep tuning the system.

## Reference materials
- US Library of Congress: https://www.loc.gov/websites/
- EU Web Archive: http://data.europa.eu/webarchive
- Mercator: A scalable, extensible web crawler (Heydon & Najork, 1999)
- Web Crawling survey (Olston & Najork): http://infolab.stanford.edu/~olston/publications/crawling_survey.pdf
- Bloom filter (Bloom, 1970): space/time trade-offs in hash coding with allowable errors
- The PageRank citation ranking (Page, Brin, Motwani, Winograd, 1998)
- Google Dynamic Rendering: https://developers.google.com/search/docs/guides/dynamic-rendering

## My open questions / follow-ups
- *(add doubts here as they come up)*
