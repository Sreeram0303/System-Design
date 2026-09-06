# Chapter 5: Design Consistent Hashing

To scale horizontally, requests/data need to be distributed **efficiently and evenly** across servers. Consistent hashing is the standard technique for this.

## The rehashing problem

Naive approach with N cache servers:
```
serverIndex = hash(key) % N
```

This works fine while N is fixed and data is evenly spread. The problem: when a server is **added or removed**, N changes — and since almost every key's `hash(key) % N` result changes too, **most keys get remapped to a different server**, not just the ones that were on the affected server. That triggers a storm of cache misses across the whole system.

**Example:** 4 servers, 8 keys hashed via `hash(key) % 4`. Take server 1 offline → now `% 3` — nearly every key's target server changes, even though only one server actually went away.

## Consistent hashing

> "Consistent hashing is a special kind of hashing such that when a hash table is re-sized, only k/n keys need to be remapped on average, where k is the number of keys and n is the number of slots. In contrast, in most traditional hash tables, a change in the number of array slots causes nearly all keys to be remapped." — Wikipedia

### Hash space and hash ring

Use a hash function (e.g. SHA-1) whose output range is huge (SHA-1: 0 to 2¹⁶⁰-1). Instead of using that range as a line, **join the two ends** to form a **ring** — hash values wrap around.

### Placing servers and keys on the ring

- **Servers** are hashed (by IP or name) onto the ring, using the *same* hash function — no modular operation this time.
- **Keys** are hashed onto the same ring.

### Server lookup

To find which server owns a key: **start at the key's position and walk clockwise** until you hit a server. That server owns the key.

Both the server and the key get their position **independently**, from hashing their own identity (`hash(server_ip)`, `hash(key)`) — a key is never "placed on" a server directly. Ownership is purely the outcome of the clockwise-walk rule. This is what makes add/remove cheap: a key's own position on the ring never moves, so changing the server set only changes *which server you hit first* for keys in one specific arc — every other key's walk lands on the exact same server it always did.

**"Clockwise" past the end wraps around.** The ring is circular, so once you pass the maximum hash value you continue from 0. In practice this isn't implemented by spinning a needle around a circle — server hashes are kept in a **sorted structure** (sorted array, or a balanced BST / `TreeMap`), and a lookup is a binary search for the smallest server hash **≥** the key's hash (`O(log n)`). If the key's hash is larger than every server's hash, the wraparound is just: the owner is the server with the **smallest** hash in the structure. More virtual nodes (below) means more entries in this structure — lookup stays `O(log n)` but with a bigger constant, plus more metadata stored.

### Adding a server

Only the keys that fall **between the new server and the previous server (going counter-clockwise from the new server)** get redistributed — to the new server. Every other key stays exactly where it was. Example: adding server 4 only moves `key0`; `key1`, `key2`, `key3` are untouched.

### Removing a server

Symmetric: only the keys that were owned by the removed server get reassigned — to the next server clockwise. Every other key is unaffected. Example: removing server 1 only remaps `key1` → server 2.

## Two problems with the basic ring approach

Consistent hashing (Karger et al., MIT) in its basic form:
1. Map servers and keys onto the ring with a uniformly-distributed hash function.
2. To resolve a key, walk clockwise to the first server.

But:

1. **Uneven partition sizes.** A "partition" is the hash-space arc between two adjacent servers. Since server placement is essentially random, some servers can end up owning a tiny sliver of the ring while a neighbor owns a huge one — e.g. if server 1 is removed, server 2's partition can become 2× the size of server 0's or server 3's.
2. **Non-uniform key distribution.** Depending on where hashing happens to place servers, most keys can cluster onto a single server while others sit nearly empty — the exact hotspot problem sharding was already vulnerable to (Chapter 1).

## Virtual nodes (the fix)

Instead of one point on the ring per physical server, represent each server with **many points** — virtual nodes / replicas (e.g. `s0_0`, `s0_1`, `s0_2` for server 0). Lookup logic is unchanged: walk clockwise to the first virtual node, then resolve it back to its physical server.

**Why it helps:** with more virtual nodes, ownership arcs get chopped into many small, scattered pieces per server instead of one big contiguous block — so the total space each *physical* server ends up owning evens out. This is a statistical effect: standard deviation (how spread-out the distribution is) shrinks as virtual node count grows. Empirically: ~100 virtual nodes → std. dev. ≈10% of the mean; ~200 → ≈5%.

**Tradeoff:** more virtual nodes = more storage/metadata needed to track them. Tune the count to fit your system.

## Finding affected keys on add/remove

- **Adding a server** (e.g. s4): walk **counter-clockwise** from the new server until you hit the previous server (e.g. s3). Every key in the arc `(s3, s4]` gets redistributed to s4.
- **Removing a server** (e.g. s1): walk **counter-clockwise** from the removed server until you hit the previous server (e.g. s0). Every key in the arc `(s0, s1]` gets redistributed to the *next* server clockwise (s2).

## What "only a fraction of keys move" actually means

When a server is removed, the keys that move are **exactly** the ones it owned — the arc between it and the previous server — nothing more, nothing less. That's a *specific, contiguous* set of keys, not a random ~1/n sample.

Whether that fraction is close to `1/n` (the "k/n keys" from the formal definition) isn't guaranteed by the basic ring — it depends on how much of the ring that particular server happened to own. This is exactly why the "two problems" above matter here: without virtual nodes, one server could own a wildly disproportionate share, so removing it might move way more or way less than `1/n` of the data. Virtual nodes make each physical server's total share converge toward `1/n`, which is what makes the formal "k/n on average" claim actually hold in practice.

**Contrast with naive `hash % N`:** there, removing one server reshuffles *nearly every key*, because N changes and that changes the modulo result even for keys that had nothing to do with the removed server. In consistent hashing, unaffected keys are structurally untouched — their clockwise walk never passed through the removed server's position in the first place.

## What consistent hashing does *not* solve

This is an easy gap to miss: consistent hashing is purely a **routing/partitioning** primitive. It answers "given this key, which server is responsible for it, right now?" — for both new writes and existing reads — and does so stably as the server set changes. **It says nothing about how the actual data gets there.** Whether that's a problem depends entirely on what's behind the ring:

- **If the ring is routing to cache servers** (the chapter's own opening example — "n cache servers"): nothing needs to be moved at all. The database is still the source of truth. When a server dies, the next request for one of its keys computes a new clockwise lookup, misses on the new server, falls through to the DB, and gets cached fresh there. "Redistributing keys" for a cache just means *which server will cache-miss next* — that's exactly why minimizing the remapped fraction matters: each remapped key is one cache miss hitting the database. Naive `hash % N` remaps almost everything at once → a miss storm hitting the DB simultaneously; consistent hashing keeps that storm small. No data-loss risk here, since the cache was never authoritative.
- **If the ring is routing to a real data store** (Dynamo, Cassandra — both listed below as production users, and neither is "just a cache"): losing a server's only copy for real would be data loss, so consistent hashing is always paired with **replication** — each key is stored on the ring-assigned server *and* the next N-1 servers clockwise (Dynamo calls this a "preference list"), not just one. When a server dies, the surviving N-1 replicas keep serving that key immediately; in the background, the system notices the key range is under-replicated and copies it from a surviving replica onto whichever server now becomes the new Nth replica — restoring full redundancy. Transient failures are often handled even more cheaply via **hinted handoff**: a neighboring server temporarily accepts writes on the dead server's behalf until it comes back.

**The clean way to carry this forward:** consistent hashing gives you the ring — a stable answer to "which node." What actually lives at each node, and how it survives that node dying (replication factor, hinted handoff, quorums, conflict resolution), is a separate layer covered properly in **Chapter 6 (Design a Key-Value Store)**. Chapter 5 hands Chapter 6 the routing primitive; it doesn't pre-solve durability.

## Wrap-up — benefits

- Minimizes key redistribution when servers are added/removed.
- Makes horizontal scaling easy — data stays evenly distributed.
- Mitigates the **hotspot key problem** (Chapter 1's "celebrity problem") by spreading data more evenly instead of concentrating it.

**Used in production by:** Amazon Dynamo's partitioning component, Apache Cassandra's cluster data partitioning, Discord, Akamai's CDN, Google's Maglev network load balancer.

## My open questions / follow-ups
- *(add doubts here as they come up)*
