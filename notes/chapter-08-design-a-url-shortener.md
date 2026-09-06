# Chapter 8: Design a URL Shortener

A classic, practical chapter — less about new distributed-systems primitives, more about *applying* what earlier chapters already built (back-of-envelope math, unique ID generation, caching, rate limiting) into one concrete system.

## Requirements

- 100 million URLs generated per day.
- Shortened URL should be as short as possible.
- Allowed characters: `0-9, a-z, A-Z` (62 characters).
- Shortened URLs are never deleted or updated (simplifying assumption).

**Two core use cases:** (1) shorten a long URL, (2) redirect a short URL back to the original.

## Back-of-envelope estimation

```
Write QPS  = 100,000,000 / 86,400 ≈ 1,157 → rounds to ~1,160
Read QPS   = 1,160 × 10 (10:1 read:write ratio) = 11,600
10-year total records = 100M × 365 × 10 = 365 billion
Storage = 365 billion × 100 bytes (avg URL length) = 36,500,000,000,000 bytes ≈ 36.5 TB
```

> **Errata caught while studying this chapter**: the book states this comes out to **365 TB**, but its own arithmetic (`365 billion × 100 bytes`) actually gives **36.5 TB**. The `365 billion` record count *already* includes the 10-year multiplier (`100M/day × 365 days × 10 years`) — multiplying by "10 years" a second time in the storage step double-counts it, effectively computing storage as if the service ran for 100 years, not 10. Worth independently re-deriving numbers rather than trusting a source at face value — see the reflection at the end of this file.

## API design

- `POST api/v1/data/shorten` — body `{longUrl}` → returns the short URL.
- `GET api/v1/shortUrl` — returns the long URL for redirection.

## URL redirecting, and the 301 vs. 302 tradeoff

| | 301 (permanent) | 302 (temporary) |
|---|---|---|
| Browser caching | Yes — subsequent clicks skip your servers entirely | No — every click routes through your service first |
| Server load | Lower (only first click hits your servers) | Higher (every click hits your servers) |
| Analytics | Poor — you only ever see the first click | Good — every click is trackable (when, source, frequency) |

**The tradeoff in one line**: 301 optimizes for infrastructure cost; 302 optimizes for data. A genuine product decision, not just a technical footnote.

## Data model

The naive high-level design keeps `<shortURL, longURL>` in an in-memory hash table — fine for explaining the idea, useless for 36+ TB of real data. The actual design stores this mapping in a **relational database** instead: a simple table of `id, shortURL, longURL`.

## The hash function — deriving the short code length

Need the smallest `n` such that `62ⁿ ≥ 365 billion` (the 10-year capacity requirement), using all 62 allowed characters per position:
```
62⁶ — not enough
62⁷ ≈ 3.5 trillion — comfortably above 365 billion
```
→ **7 characters.**

### Approach 1 — hash + collision resolution

Hash the long URL (CRC32, MD5, SHA-1) and take the first 7 characters. Problem: even the shortest of these (CRC32) produces more than 7 characters, so truncating is required — and truncating creates real collision risk (two different long URLs landing on the same 7-character prefix).

**Collision resolution**: if a candidate short code already maps to a *different* long URL, recursively append a predefined string to the original URL and re-hash, repeating until a free code is found.

**The cost**: a database lookup on *every* shortening request just to check for a collision. Mitigated the same way Chapter 6's read path handles this exact shape of problem — a **bloom filter** cheaply says "definitely not taken" (skip the DB check) or "might be taken" (worth a real lookup).

### Approach 2 — base 62 conversion (the one chosen)

Take a **globally unique integer ID** (Chapter 7's ID generator) and convert it directly to base 62. Since the ID is already guaranteed unique, the resulting code is guaranteed unique too — **no collisions possible, no DB check needed** to detect them.

**Worked example** — convert decimal `11157` to base 62:
```
11157 = 2 × 62² + 55 × 62¹ + 59 × 62⁰
      = 2 × 3844 + 55 × 62 + 59 × 1
      = 7688 + 3410 + 59 = 11157 ✓

digit → character mapping:
  0-9   → '0'-'9'
  10-35 → 'a'-'z'
  36-61 → 'A'-'Z'

2  → '2'
55 → 55-36=19 → 'A'+19 = 'T'
59 → 59-36=23 → 'A'+23 = 'X'

→ "2TX"
```

### Why base 62 wins

| | Hash + collision resolution | Base 62 conversion |
|---|---|---|
| Collisions possible? | Yes — needs detection + resolution logic | No — uniqueness inherited structurally from the ID generator |
| Needs a DB check per write? | Yes (mitigated by bloom filter) | No |
| Extra dependency | None | Chapter 7's unique ID generator |
| Short code length | Fixed | Grows slowly as the ID grows (bounded while ID < 62⁷) |

**Full circle**: Chapter 7's unique ID generator exists largely *so that* this chapter can use it here — base 62 avoids collision handling entirely by offloading the uniqueness guarantee onto an already-solved problem.

## The full shortening flow

```
1. longURL comes in
2. Check DB: has this longURL been shortened before?
3. If yes → return the existing shortURL
4. If no → get a new unique ID from the ID generator (Ch7)
5. Convert that ID to a shortURL via base 62
6. Store {id, shortURL, longURL} as a new row
```
Book's example: ID `2009215674938` → base62 → `"zn9edcu"`.

## The redirecting flow

```
1. User clicks https://tinyurl.com/zn9edcu
2. Load balancer routes to a web server
3. Check cache first — hit? return longURL immediately
4. Miss? fetch from DB (not found = likely an invalid short URL)
5. Return longURL, browser redirects
```
Since reads outnumber writes 10:1, caching `<shortURL, longURL>` is a direct payoff of Chapter 1's cache-tier reasoning.

## Wrap-up talking points — all callbacks to earlier chapters

- **Rate limiting** (Ch4) — guards the shortening endpoint against abuse.
- **Web server scaling** (Ch1) — stateless web tier, scale by adding/removing servers.
- **Database scaling** (Ch1, 5, 6) — replication and sharding once a single DB can't hold 36+ TB.
- **Analytics** — a legitimate business reason to prefer 302 over 301.
- **Availability, consistency, reliability** (Ch1) — general principles apply here too.

## What back-of-envelope estimation actually tells an engineer

Prompted by catching the book's own 36.5 TB vs. 365 TB error above — the estimation process isn't a warm-up ritual, it's the translation layer between a vague requirement and a concrete engineering spec:

1. **It falsifies bad designs before you build them** — the naive "everything in a hash table" idea dies the instant the 36.5 TB number appears; that's *why* the design pivots to a relational database, not a stylistic choice.
2. **It tells you where to spend engineering effort** — the 10:1 read:write skew is *why* the redirect flow gets a cache layer and the shortening flow doesn't.
3. **It can directly derive a design parameter** — "365 billion capacity" + "62 characters" mathematically *produces* "7-character codes," not just validates a guess.
4. **It's the guard against over-engineering in both directions** — small numbers would tell you none of this machinery (ID generator, sharding, caching) is justified at all; big numbers are what earn the right to reach for it.
5. **It forces assumptions into the open** — "100 bytes/URL," "10:1 ratio," "10 years" are all assumptions, written down so they're checkable and revisable later, not just facts.
6. **It's a check on your own reasoning** — the 36.5 TB vs. 365 TB catch only happened *because* the calculation was written out step by step. Skip that discipline and a 10× error sails through unnoticed.

## Reference materials
- A RESTful Tutorial: https://www.restapitutorial.com/index.html
- Bloom filter: https://en.wikipedia.org/wiki/Bloom_filter

## My open questions / follow-ups
- *(add doubts here as they come up)*
