# Chapter 10: Design a Notification System

A notification alerts a user to something important — breaking news, a product update, a payment confirmation. **Three formats**: mobile push notification, SMS message, email — each with a completely different delivery mechanism and third-party dependency.

## Requirements

Supports push/SMS/email; a **soft real-time** system (fast delivery preferred, but a slight delay under heavy load is acceptable — a deliberately looser latency bar than, say, the rate limiter's "must not slow down HTTP response time"); supports iOS, Android, and desktop/laptop; triggered either by client applications or scheduled server-side; users can opt out; **volume**: 10 million push notifications, 1 million SMS, 5 million emails per day.

## How each notification type actually works

| Type | Key components | Delivery service |
|---|---|---|
| **iOS push** | Provider (builds the request), device token (unique per-device identifier), payload (JSON) | Apple Push Notification Service (APNS) |
| **Android push** | Same shape as iOS | Firebase Cloud Messaging (FCM) |
| **SMS** | — | Third-party: Twilio, Nexmo |
| **Email** | — | Third-party: Sendgrid, Mailchimp |

**Every delivery path routes through a third-party service you don't control** — a fundamentally different shape of problem than most earlier chapters, where the whole stack was yours to control. This is exactly why "a third-party service might be unavailable" (e.g., FCM doesn't work in China; alternatives like Jpush/PushY are used there) is a first-class design concern, and why **extensibility** — plugging/unplugging a provider without a system redesign — matters specifically here.

## Contact info gathering flow

When a user installs the app or signs up, API servers collect contact info and persist it. **Schema split matters**: email/phone live in the **user table**, but device tokens live in a **separate device table**, since a user can have **multiple devices**. Consequence: "send a notification to a user" can mean fanning it out to *several* device tokens at once, not just one.

## Initial high-level design, and its three problems

**Components**: Services 1-N (anything that triggers a notification) → a single Notification server (builds payloads) → Third-party services → iOS/Android/SMS/Email devices.

Three problems, each a known failure pattern from earlier chapters recurring in a new context:

1. **Single point of failure** — one notification server. Same SPOF pattern Chapter 1's load balancer and Chapter 6's decentralized architecture exist to eliminate.
2. **Hard to scale** — DB, cache, and processing logic all bundled into one server, can't scale independently. Violates Chapter 1's "separate tiers scale independently" lesson.
3. **Performance bottleneck** — building payloads and waiting on third-party responses is slow; handling it synchronously means peak load in one area (e.g., an SMS flood) can stall unrelated notifications too.

## Improved high-level design — decoupling the pipeline

| Problem | Fix | Origin |
|---|---|---|
| SPOF | Move DB/cache out of the notification server; multiple notification servers behind a load balancer, auto-scaled | Ch1 — stateless web tier + load balancing |
| Hard to scale | DB, cache, notification servers now separate tiers, scaled independently | Ch1 — separate tiers |
| Performance bottleneck | Message queues between "build the notification" and "deliver it" | Ch1's message queue |

**Each notification type gets its own distinct queue** (iOS PN queue, Android PN queue, SMS queue, Email queue) — a deliberate failure-isolation choice. If FCM has an outage, only the Android queue backs up; SMS and email keep draining normally. A single shared queue would turn one third-party outage into a systemic one.

**Notification servers**: expose internal-only APIs (prevents spam from untrusted callers), validate input, fetch rendering data from cache/DB, push events onto the correct queue. **Workers**: pull events from queues, call the actual third-party services.

### The 6-step flow

```
1. A service calls the notification server's API
2. Notification server fetches user info, device token, settings from cache/DB
3. The event is pushed to the correct queue (e.g., an iOS push → iOS PN queue)
4. Workers pull events from their queue
5. Workers call the corresponding third-party service
6. The third-party service delivers to the actual device
```
The notification server's job ends at step 3 — it never waits on a third-party response. Slow, unpredictable third-party latency is isolated to the workers and can never stall the API layer.

## Reliability

### Preventing data loss

**Hard requirement**: notifications can be delayed or reordered, but **never lost**. Fix: persist notification data to a **notification log** database as part of the send process, plus a retry mechanism. Same durability instinct as Chapter 6's write path — write the durable record first, before attempting the actual unreliable operation.

### Will a recipient get exactly one copy? No.

**"Exactly-once delivery" is not achievable** as a network-level guarantee. If a worker sends a notification and never receives an ack, it can't distinguish *"never delivered"* from *"delivered fine, but the ack was lost."* Both look like silence. The only safe move is to retry — but if it was actually the second case, the recipient now gets a duplicate.

**The two honest baseline options**: **at-most-once** (never retry on ambiguity — never duplicate, but risk losing messages) or **at-least-once** (always retry on ambiguity — never lose, but risk duplicates). The requirement ("never lost, delay/duplicate tolerable") picks the answer: **at-least-once, plus a deduplication layer on top**.

**Dedup mechanism**: check a notification event's ID against what's already processed — seen before → discard, new → send. Same shape of problem as Chapter 9's "Content Seen?"/"URL Seen?" components, applied to notification events instead of pages.

> **The pattern worth naming**: this is the same move Chapter 6 made with eventual consistency — don't try to *prevent* an imperfection outright (lost writes there; lost notifications here); pick a simple, robust base guarantee that allows a specific, *bounded* imperfection (stale reads there; duplicate sends here), then bolt on a thin reconciliation layer (vector clocks there; event-ID dedup here) to clean it up after the fact. Preventing the imperfection at the source is usually far more expensive than allowing it and cleaning up afterward.

## Additional components

- **Notification templates** — avoid building millions of similar notifications from scratch. Example: `BODY: You dreamed of it. We dared it. [ITEM NAME] is back — only until [DATE].` Benefits: consistent format, fewer manual errors, faster iteration.
- **Notification settings** — `{user_id, channel, opt_in}`. Check opt-in status for the *specific channel* before sending anything — a user might allow email but disable push.
- **Rate limiting** — Chapter 4's rate limiter, but **inverted**: there it protected the *server* from too many client requests; here it protects the *user* from too many notifications sent *by your own services*. Over-notifying has a real cost — users who feel spammed disable notifications entirely, killing the channel for every future message.
- **Retry mechanism** — failed sends go back on the queue; if failures persist past a threshold, **alert developers** rather than retrying silently forever.
- **Security** — `appKey`/`appSecret` pairs authenticate which clients can call the push APIs at all, preventing unauthorized senders from spamming users through your infrastructure.
- **Monitor queued notifications** — a growing queue depth means workers aren't keeping up; add more. The *runtime* counterpart to the back-of-envelope estimate done at design time — queue depth is how you detect live that reality has outrun the original capacity plan.
- **Events tracking** — open rate, click rate, engagement, fed to an analytics service — genuinely useful for telling whether targeting/rate limiting is working, not just a vanity metric.

## Wrap-up

The final design layers authentication and rate-limiting onto the notification servers, adds a retry mechanism for failures, uses templates for consistent notification creation, and adds monitoring/tracking for system health and future tuning. Core themes: **reliability** (robust retries), **security** (verified clients only), **respecting user settings** (check opt-in before sending), and **rate limiting** (protecting the user experience, not just the infrastructure).

## Reference materials
- Twilio SMS: https://www.twilio.com/sms
- Nexmo SMS: https://www.nexmo.com/products/sms
- Sendgrid: https://sendgrid.com/
- Mailchimp: https://mailchimp.com/
- You Cannot Have Exactly-Once Delivery: https://bravenewgeek.com/you-cannot-have-exactly-once-delivery/
- Security in Push Notifications: https://cloud.ibm.com/docs/services/mobilepush?topic=mobile-pushnotification-security-in-push-notifications

## My open questions / follow-ups
- *(add doubts here as they come up)*
