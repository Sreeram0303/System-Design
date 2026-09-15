# Chapter 11: Design a News Feed System

"News feed: the constantly updating list of stories... status updates, photos, videos, links, app activity, and likes from people, pages, and groups you follow" (Facebook's own definition). Same question shape as designing Instagram's feed or Twitter's timeline.

## Requirements — the exact example from Chapter 3

- Both mobile and web.
- Core features: make a post, see friends' feed.
- **Reverse-chronological**, not ranked — avoids needing a whole scoring/ML subsystem.
- **5,000 max friends** — bounded, not unlimited followers.
- **10 million DAU**.
- Feed can contain images and video, not just text.

The friend-count number isn't a "scale" question — it's a **fan-out shape** question (Chapter 3's category 3): it determines whether pushing a post to every friend's feed at write time is cheap and bounded, or risks blowing up for outliers. A hard cap of 5,000 looks safe at first glance — topic "Fanout deep dive" below revisits whether it actually is.

## The two APIs

**Publish** — `POST /v1/me/feed` — params: `content`, `auth_token`.
**Retrieve** — `GET /v1/me/feed` — params: `auth_token`.

Deliberately minimal — no ranking parameters to expose, because there's no ranking algorithm behind a reverse-chronological feed. The API's simplicity is a direct, visible consequence of the "keep it simple" requirement decided upfront.

## Feed publishing flow — the write path

`User → Load Balancer → Web Servers → {Post Service, Fanout Service, Notification Service}`.

- **Post service** — persists the new post to database and cache.
- **Fanout service** — pushes the post into friends' news feed caches (see deep dive below).
- **Notification service** — informs friends new content is available, sends push notifications.

Same write-path/read-path split as Chapter 6, 8, and 9 — publishing is the write path; news feed building (next) is its read-path counterpart.

## News feed building flow — the read path (high-level)

`User → Load Balancer → Web Servers → Newsfeed Service → Newsfeed Cache`. At this level of detail, reading a feed is a single cache lookup — deliberately simple, because the real complexity is pushed backward (into fanout, done ahead of time) or downstream (into hydration at retrieval, covered below).

## Fanout deep dive — the payoff of Chapter 3's fan-out question

**Fanout** = delivering a post to all of a user's friends.

### Fanout on write (push model)

Pre-compute the feed at write time — a new post is pushed directly into every friend's cache immediately.
- **Pros**: real-time delivery; fast reads (feed is already assembled).
- **Cons**: the **hotkey problem** — a user with many friends makes fetching/writing to all of them slow and resource-intensive. Also wastes compute pre-generating feeds for inactive users who may never read them before they're stale.

### Fanout on read (pull model)

Assemble the feed on demand at read time — recent posts pulled from friends only when a user loads their home page.
- **Pros**: no wasted work on inactive users; no hotkey problem (no write-time burst at all).
- **Cons**: reads become slow — every read now assembles and merges content live.

### The hybrid design — the actual answer

**Push for the majority of users** (fast reads matter most for typical accounts); **pull for celebrities/high-follower accounts** — their followers pull that content on-demand at read time, merged in with everything arriving via push. This sidesteps the worst case of both models. **Consistent hashing (Ch5)** helps distribute whatever fanout load still needs to happen more evenly across shards, mitigating the hotkey problem rather than concentrating it.

> **Worth being precise**: the requirements capped friends at 5,000 — bounded, but the deep dive still flags a hotkey problem. "Bounded" doesn't automatically mean "small enough to ignore" — 5,000 synchronous fanout writes per post, across a very active account and 10M DAU of overall volume, is still large relative to the write budget. Same lesson as Chapter 9's politeness problem in a different costume: an aggregate system budget needs explicit protection from any single outlier, and a cap alone doesn't remove that need.

## The fanout service, step by step

1. **Fetch friend IDs from a graph database** — purpose-built for relationship data, a different tool than the relational/KV stores from earlier chapters because the access pattern (traverse a relationship graph) fits it naturally.
2. **Get friends' info from the user cache, filter by settings** — muted friends, or posts selectively hidden from certain people, get filtered out here, before anything is fanned out.
3. **Send the friends list + new post ID to a message queue** — not processed inline with the publish request. Fanning out to thousands of friends shouldn't block the "post published" response; queue it and let workers handle it asynchronously (Chapter 1's message queue lesson).
4. **Fanout workers pull from the queue and write into the news feed cache** — a `<post_id, user_id>` mapping, appended per relevant friend.
5. **Only IDs are stored, never full objects.** Storing complete post/user data per feed entry at this scale would be enormous. A configurable per-user size limit keeps the cache small — since almost nobody scrolls back thousands of posts, this keeps the cache miss rate low without over-storing.

## News feed retrieval — hydration

```
1. User requests /v1/me/feed
2. Load balancer → web servers
3. Web servers call the newsfeed service
4. Newsfeed service fetches a list of post IDs from the newsfeed cache
5. Fetch full user and post objects (username, profile picture, content,
   images) from the user cache and post cache — "hydration"
6. Return the fully hydrated feed as JSON
```
**Media (images/video) is served from a CDN** — same reasoning as Chapter 1's CDN section, applied to feed content.

**Hydration is the direct payoff — and cost — of storing only IDs** in the newsfeed cache: storage stays small and cheap, but retrieval needs an extra assembly step to reconstruct something renderable. A clean, minimal instance of a recurring tradeoff seen throughout this book: compact storage traded for extra work at read time (same shape as Chapter 6's bloom filter needing a follow-up check, or Chapter 8's base-62 code needing a DB lookup for the real long URL). Cheap to store rarely comes free — the cost usually just moves to whoever reads it back.

## Cache architecture — five layers, not one

| Layer | Stores | Access pattern |
|---|---|---|
| **News Feed** | `<post_id, user_id>` IDs | Very read-heavy, tiny entries, critical path of every feed load |
| **Content** | Full post data | Read-skewed toward popular/recent posts — hot/cold split within the layer |
| **Social Graph** | Friend/follow relationships | Read-heavy, but changes far less often than posts |
| **Action** | Likes/replies on a post | Frequent writes *and* reads — different balance than mostly-write-once layers |
| **Counters** | Like/reply/follower/following counts | Extremely high write volume relative to entry size |

**Why split instead of one shared cache**: same instinct as Chapter 1's separate tiers and Chapter 10's per-type queues — bundling genuinely different access patterns (read-heavy vs. write-heavy, tiny vs. large, fast-changing vs. slow-changing) into one shared resource means none of them can be tuned or scaled independently.

## Wrap-up — scaling talking points

**Database scaling:**
- Vertical vs. horizontal scaling
- SQL vs. NoSQL
- Master-slave replication
- Read replicas
- Consistency models
- Database sharding

**Other talking points:**
- Keep the web tier stateless
- Cache data as much as possible
- Support multiple data centers
- Decouple components with message queues
- Monitor key metrics — e.g. QPS during peak hours, latency while users refresh their feed

Every one of these is a direct callback to a chapter already covered in this series (Ch1, Ch5, Ch6) — this chapter's real contribution isn't new infrastructure vocabulary, it's showing how all of it comes together to solve one concrete, familiar product.

## Reference materials
- How News Feed Works: https://www.facebook.com/help/327131014036297/
- Friend of Friend recommendations with Neo4j and SQL Server: http://geekswithblogs.net/brendonpage/archive/2015/10/26/friend-of-friend-recommendations-with-neo4j.aspx

## My open questions / follow-ups
- *(add doubts here as they come up)*
