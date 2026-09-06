# Chapter 3: A Framework for System Design Interviews

No one expects you to design a real production system (Google Search, etc.) in 45 minutes — real systems take thousands of engineers years to build. The interview instead simulates two co-workers collaborating on an ambiguous, open-ended problem. **The process matters more than the final design.**

## What interviewers are actually evaluating

Not just technical design skill. They're reading for:
- Ability to **collaborate** and work under pressure.
- Ability to **resolve ambiguity** constructively.
- Ability to **ask good questions** — many interviewers specifically look for this.

**Red flags to avoid:**
- **Over-engineering** — chasing design purity while ignoring tradeoffs and their compounding real-world cost.
- Narrow-mindedness, stubbornness, refusing feedback.

> Don't be like Jimmy — the student who always answers fast without thinking. Answering quickly with no clarification isn't a bonus; the interview isn't a trivia contest and there's no single right answer.

## The 4-step framework

### Step 1 — Understand the problem and establish design scope (~3-10 min)

Slow down. Ask questions before proposing anything. If the interviewer asks you to make an assumption instead of answering directly, **write the assumption down** — you'll need it later.

**Questions worth asking:**
- What specific features are we building?
- How many users does the product have?
- How fast is the company expecting to scale — 3 months, 6 months, a year out?
- What's the existing tech stack? What can we leverage instead of building from scratch?

**Example — clarifying "design a news feed system":**
| Candidate asks | Interviewer answers |
|---|---|
| Mobile, web, or both? | Both |
| Most important features? | Make a post, see friends' news feed |
| Reverse-chronological, or weighted/ranked order? | Keep it simple — reverse chronological |
| How many friends can a user have? | 5,000 |
| Traffic volume? | 10 million DAU |
| Text only, or media too? | Images and videos too |

**Why these questions matter — each one collapses a specific axis of the design space, before any box gets drawn:**

| Question | Design axis it pins down | What changes with a different answer |
|---|---|---|
| Mobile, web, or both? | Client surface | Both forces a platform-agnostic API instead of server-rendered pages, plus push notifications and bandwidth-conscious media delivery for mobile |
| Most important features? | **Scope** | Without this, "news feed system" is unbounded — comments, likes, notifications, search could all be in scope. This pins it to exactly two flows: publish + retrieve |
| Chronological or ranked? | **Behavior / complexity** of an existing feature | Ranked (EdgeRank-style) needs a whole scoring/ML subsystem — arguably a separate interview by itself. Chronological is just "sort by timestamp." Same feature, order-of-magnitude difference in what gets built |
| How many friends can a user have? | **Fan-out / relationship shape** | Bounded (5,000) makes fan-out-on-write cheap and predictable — one post, ≤5,000 writes. Unbounded (Twitter-style followers) breaks that for celebrities, forcing a hybrid push/pull model. This is what decides the *write-path architecture*, covered in depth in Chapter 11 |
| Traffic volume? | **Scale inputs** | Feeds Chapter 2's back-of-envelope math directly → QPS/storage estimates → whether sharding, replication, or a cache tier (Chapter 1) are needed. At 10M DAU × up to 5,000 friends/post, synchronous fan-out would spike hard, which is why real designs push it through an async message queue instead |
| Text only, or media too? | **Data shape** | Media needs object storage + a CDN (Chapter 1) and probably an async transcoding/thumbnailing pipeline; text alone could live in a database. Storage math changes by orders of magnitude — same asymmetry as Chapter 2's Twitter example (140 bytes of text vs. 1 MB of media per post) |

**A sharper set of categories to ask through** (refines the four bullets above into buckets that don't overlap):
1. **Scope** — what does the system do, at minimum?
2. **Behavior / complexity** — for a feature that exists, how sophisticated does it need to be?
3. **Fan-out / relationship shape** — how many things does *one* action touch? (bounded vs. unbounded). This is a different question from raw user count — it's what actually decides push vs. pull architecture.
4. **Scale inputs** — the raw numbers (DAU, actions/user/day, size/action) that feed the Chapter 2 math. The trigger for "this needs a big-scale design" isn't a headcount threshold to eyeball — it's what the QPS/storage numbers come out to once you actually run the math.

### Step 2 — Propose high-level design and get buy-in (~10-15 min)

Goal: reach agreement with the interviewer on a high-level blueprint — treat them as a teammate, not an examiner.

- Sketch box diagrams: clients, APIs, web servers, data stores, cache, CDN, message queue, etc.
- Do back-of-the-envelope math to sanity-check the blueprint against scale constraints (Chapter 2). Say out loud when you're about to do this.
- Walk through a few concrete use cases — this surfaces edge cases you haven't considered yet.
- **How much low-level detail (API endpoints, DB schema) belongs here?** Depends on the problem's scope — too low-level for "design Google Search," entirely fair game for "design the backend of a multiplayer poker game." Read the room / ask.

**Example — news feed, high level:** splits into two flows —
- **Feed publishing** — a post is written to cache/DB, then fanned out into friends' news feeds.
- **News feed building** — aggregate friends' posts in reverse-chronological order.

### Step 3 — Design deep dive (~10-25 min)

By now you and the interviewer should have: agreed on scope, sketched the high-level blueprint, gotten feedback on it, and picked up hints on where to go deeper.

Work together to **identify and prioritize** which components deserve a deep dive — it varies interview to interview:
- Senior-level interviews often push toward performance characteristics: bottlenecks, resource estimation.
- URL shortener → the hash function design is the interesting part.
- Chat system → latency reduction and online/offline status are the interesting parts.

**Manage your time.** It's easy to get pulled into details that don't actually demonstrate ability (e.g. going deep on Facebook's EdgeRank ranking algorithm burns time without proving you can design a scalable system).

**Example — news feed deep dive:** two use cases get detailed designs — feed publishing, and news feed retrieval.

### Step 4 — Wrap up (~3-5 min)

- Identify bottlenecks and discuss potential improvements. **Never claim the design is perfect** — there's always something to improve, and this is your chance to show critical thinking.
- Recap the design, especially useful if you proposed multiple approaches across a long session.
- Talk about error cases: server failure, network loss, etc.
- Mention operational concerns: metrics/error-log monitoring, rollout strategy.
- Discuss the next scale curve — e.g. "this design supports 1M users; what changes to get to 10M?"
- Propose further refinements you'd make given more time.

## Time allocation (45-minute interview, rough guide)

| Step | Time |
|---|---|
| 1 — Understand the problem, establish scope | 3–10 min |
| 2 — High-level design, get buy-in | 10–15 min |
| 3 — Design deep dive | 10–25 min |
| 4 — Wrap up | 3–5 min |

*(Actual distribution depends on the problem's scope and the interviewer's focus — this is a rough default, not a rule.)*

## Dos and Don'ts

**Dos**
- Always ask for clarification — don't assume your assumption is correct.
- Understand the actual requirements before designing (a scrappy startup's solution ≠ an established company's solution at millions of users).
- Think out loud — communicate constantly.
- Suggest multiple approaches where relevant.
- Once the blueprint is agreed, go into details on each component — most critical component first.
- Bounce ideas off the interviewer like a teammate.
- Never give up.

**Don'ts**
- Don't be unprepared for typical interview questions.
- Don't jump to a solution before clarifying requirements/assumptions.
- Don't go deep on one component early — high-level first, then drill down.
- Don't hesitate to ask for hints if stuck.
- Don't think in silence — communicate.
- Don't assume you're done once you've given a design — you're done when the interviewer says so. Ask for feedback early and often.

## My open questions / follow-ups
- *(add doubts here as they come up)*
