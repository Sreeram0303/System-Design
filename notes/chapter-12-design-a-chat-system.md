# Chapter 12: Design a Chat System

"Chat app" hides real product diversity: one-on-one apps (Messenger, WhatsApp), group-focused office chat (Slack), large-group/low-latency voice chat (Discord). Designing for the wrong one wastes the whole interview on a mismatch — Chapter 3's scoping lesson with unusually high stakes.

## Requirements

Supports **both** 1-on-1 and group chat, mobile and web, **50 million DAU**, group chat capped at **100 members**, text-only, messages under 100,000 characters, no end-to-end encryption required (for now), chat history stored **forever**.

**Feature focus**: low-latency 1-on-1 chat, small group chat, online presence, multi-device support (same account logged in on several devices at once), push notifications.

## The core protocol problem — how does a server push to a client?

**Sender side is easy**: HTTP works fine, especially with keep-alive to avoid repeated TCP handshakes.

**Receiver side is hard** — HTTP is fundamentally client-initiated. Three progressively better workarounds:

- **Polling** — client periodically asks "anything new?" Wasteful; most polls return "no."
- **Long polling** — client holds a connection open until new messages exist or a timeout hits. Problems: sender/receiver may land on different (stateless, round-robin-balanced) servers; a server can't reliably tell if a client disconnected; inefficient for quiet users who still trigger periodic reconnects.
- **WebSocket** — starts as HTTP, upgrades via handshake into a **persistent, bidirectional** connection, riding on port 80/443 so it survives most firewalls. Since it's bidirectional anyway, using it for both directions simplifies the design, at the cost of needing careful server-side connection management.

**The one-line summary**: polling wastes resources asking too often; long polling is a clever-but-leaky workaround for HTTP's client-initiated nature; WebSocket actually inverts who can initiate — and that inversion is why the chat service, uniquely in this book, has to be genuinely **stateful**.

## High-level design

- **Stateless services** — login, signup, profile, behind a load balancer routing by request path. Includes **service discovery** (gives a client a list of chat-server hostnames to connect to — deep dive below).
- **Stateful service — the chat service.** A client holds a persistent WebSocket connection to one specific chat server and normally doesn't switch away from it while that server stays up.
- **Third-party integration — push notifications.** Direct callback to Chapter 10's already-designed notification system.

> **Worth pausing on**: since Chapter 1, "keep the web tier stateless" has been close to a universal rule. The chat service is a genuine, structural exception — forced by the WebSocket decision, not a lapse in discipline. A WebSocket connection is a real, open socket sitting in one specific machine's memory; you can't load-balance individual messages across an interchangeable pool the way you can with plain HTTP requests.

## The scalability reality check

```
1,000,000 concurrent users × 10 KB per connection ≈ 10 GB
```
A single modern server could genuinely hold this much connection memory — the math checks out. **Yet single-server is still a red flag.** Back-of-envelope estimation tells you whether a *quantity* is physically feasible; it does not tell you whether the resulting *architecture* is sound. The reasons single-server fails here (single point of failure, no scaling headroom, no maintenance without disconnecting everyone) never show up in a capacity calculation at all.

It's fine to *start* the conversation with a single-server sketch as a stepping stone — as long as it's explicit that it's a starting point — then evolve into the real multi-server design: **chat servers** (send/receive), **presence servers** (online/offline status), **API servers** (stateless features), **notification servers** (Ch10), and a **key-value store** for chat history.

## Storage — relational vs. key-value, and message IDs

**Generic data** (profile, settings, friends list) → relational databases, replicated and sharded.

**Chat history data** — a distinct workload: enormous volume (Messenger + WhatsApp process 60 billion messages/day), strong recency bias, but genuine random-access needs too (search, jump to a mention), and a read:write ratio **≈ 1:1** — notably different from most systems in this book (URL shortener ≈10:1, news feed skews read-heavy too).

**Why key-value stores win**: easy horizontal scaling, low latency; relational DBs handle the "long tail" poorly as B-tree-style indexes grow large; real-world validation (Facebook Messenger uses HBase, Discord uses Cassandra — the same Cassandra that was one of Chapter 6's three reference systems).

**Data models**: 1-on-1 chat uses `message_id` as primary key (never `created_at` — two messages can share a timestamp). Group chat uses composite key `(channel_id, message_id)`, with `channel_id` as the **partition key**, since every group query operates within one channel — a direct application of Chapter 1/5/6's "pick a partition key matching your query pattern."

**Message ID generation, three approaches**:
1. `auto_increment` — doesn't work; NoSQL stores don't offer it (echoes Chapter 7's opening problem).
2. A global 64-bit sequence generator (Snowflake, Ch7) — fully solved, drop it in.
3. A **local** sequence generator — unique only *within* a group/channel.

> **The generalizable lesson**: message ordering only ever matters within one conversation — never across unrelated ones. Relaxing "globally unique" to "unique within this partition" avoids needing Snowflake's full datacenter/machine-ID machinery. Before reaching for a fully-solved hard problem from an earlier chapter, check whether the actual requirement needs *global* uniqueness or just uniqueness within an already-bounded scope — the weaker requirement is often dramatically cheaper.

## Service discovery — a different kind of load balancing

**Purpose**: recommend the best chat server for a client, based on location and server capacity. **Apache Zookeeper** is the standard tool.

```
1. User A tries to log in
2. Load balancer routes the login request to API servers
3. Backend authenticates; service discovery picks the best chat server
   (e.g. server 2), returns its info to User A
4. User A connects to chat server 2 via WebSocket
```

**How this differs from Chapter 1's load balancer**: a regular load balancer decides fresh on every request — a slightly wrong pick costs little, since the next request can go elsewhere. Service discovery here makes **one decision per WebSocket connection**, sticking for the connection's entire lifetime (potentially hours) — a bad pick isn't self-correcting. It also solves a similar-sounding problem to Chapter 6's gossip protocol ("which nodes exist and are healthy") via a different architecture: Zookeeper is a centralized coordination service, where gossip is fully decentralized — appropriate here because clients need one definitive, authoritative answer rather than an eventually-consistent view.

## Message flows

### 1-on-1 chat flow
```
1. User A sends a message to Chat server 1
2. Chat server 1 gets a message ID from the ID generator
3. Chat server 1 sends the message to a "message sync queue"
4. The message is stored in the key-value store
5a. User B online → forward to Chat server 2 (where B is connected)
5b. User B offline → push notification via PN servers (Ch10)
6. Chat server 2 forwards the message to User B over B's WebSocket
```
Persistence (step 4) happens independently of live delivery (5a/5b) — the same durability-first instinct as Chapter 6's write path and Chapter 10's notification log: save the durable record first, then attempt best-effort live delivery.

### Multi-device sync

Each device tracks its own **`cur_max_message_id`** — a high-water mark for what it's already synced. A message counts as new if the recipient matches and its ID exceeds that device's mark. This works cheaply *because* message IDs are sortable by time (topic on message IDs above) — sortability, not just uniqueness, is what enables a single scalar bookmark instead of per-message tracking.

### Small group chat flow

A message from A in a 3-member group gets **copied into each other member's own inbox** (message sync queue). Clean for small groups: sync stays simple (check only your own inbox), and copying to a small member count isn't expensive. Explicitly scoped to the 100-member cap for this reason (WeChat caps at 500 for the same reason).

> **The parallel worth naming**: copying a message into every member's inbox on write is structurally identical to Chapter 11's fanout-on-write for news feeds, and has the exact same limitation — one write triggering N copies doesn't scale once N gets large.

## Online presence

**Three triggers**: login (WebSocket established → status + `last_active_at` saved), logout (status → offline), and **disconnection** — the hard case, since naive immediate-offline-on-disconnect causes rapid flapping from ordinary network flicker.

**The heartbeat fix**: client sends a heartbeat periodically (e.g. every 5s); if none arrives within a threshold window (e.g. 30s), *then* the status flips to offline. This is a **debouncing pattern**: don't react to one noisy signal, wait for it to persist. The same underlying move as Chapter 6's gossip protocol requiring two independent sources before declaring a node down.

**Fanout via pub-sub, modeled per relationship**: each friend *pair* gets its own channel (A-B, A-C, A-D). A status change publishes to all of A's channels at once; each friend, subscribed to their shared channel, receives it over WebSocket. This is a structurally different model than the group-chat inbox (one queue per *person*) — here it's one channel per *edge* in the social graph, because a pairwise status is naturally relationship-scoped rather than belonging to one person's timeline.

**The scaling limit — recurring pattern, third appearance**: fine for small groups (WeChat's 500-member cap uses the same approach); a 100,000-member group's status change would generate 100,000 events. Fix: stop pushing proactively for large groups — pull status only when a user enters the group or refreshes their friend list. **Identical move to Chapter 11's fanout-on-read fix for celebrities** — whenever a write threatens to fan out to an unbounded number of consumers, the fix is never "push harder," it's "stop pushing to the outliers, let them pull instead."

## Wrap-up talking points

- **Media files** (photos, videos) — significantly larger than text; compression, cloud storage, thumbnails become relevant.
- **End-to-end encryption** — only sender and recipient can read messages (WhatsApp supports this).
- **Client-side message caching** — reduces data transfer between client and server.
- **Improve load time** — Slack built a geographically distributed edge cache (Flannel) for user/channel data.
- **Error handling**:
  - **Chat server failure** — with hundreds of thousands of persistent connections on one server, Zookeeper reassigns affected clients to a new server on failure.
  - **Message resend mechanism** — retry and queueing, the same pattern as everywhere else in this book that needs at-least-once delivery.

## Reference materials
- Erlang at Facebook: https://www.erlang-factory.com/upload/presentations/31/EugeneLetuchy-ErlangatFacebook.pdf
- Messenger and WhatsApp process 60 billion messages a day: https://www.theverge.com/2016/4/12/11415198/facebook-messenger-whatsapp-number-messages-vs-sms-f8-2016
- Long tail: https://en.wikipedia.org/wiki/Long_tail
- The Underlying Technology of Messages (HBase): https://www.facebook.com/notes/facebook-engineering/the-underlying-technology-of-messages/454991608919/
- How Discord Stores Billions of Messages (Cassandra): https://blog.discordapp.com/how-discord-stores-billions-of-messages-7fa6ec7ee4c7
- Apache ZooKeeper: https://zookeeper.apache.org/
- End-to-end encryption: https://faq.whatsapp.com/en/android/28030015/
- Flannel: An Application-Level Edge Cache to Make Slack Scale: https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale-b8a6400e2f6b

## My open questions / follow-ups
- *(add doubts here as they come up)*
