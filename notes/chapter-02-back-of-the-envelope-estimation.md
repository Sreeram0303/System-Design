# Chapter 2: Back-of-the-Envelope Estimation

> "Back-of-the-envelope calculations are estimates you create using a combination of thought experiments and common performance numbers to get a good feel for which designs will meet your requirements." — Jeff Dean, Google Senior Fellow

Interviewers use this to test **problem-solving process**, not precision. Three building blocks: power of two, latency numbers, and availability numbers.

## 1. Power of two

Data volume in distributed systems always reduces to bytes, so it's worth having powers of 2 memorized cold.

| Power | Approx value | Bytes |
|---|---|---|
| 2⁷ | 128 | |
| 2⁸ | 256 | |
| 2¹⁰ | 1 thousand | 1 KB |
| 2¹⁶ | | 64 KB |
| 2²⁰ | 1 million | 1 MB |
| 2³⁰ | 1 billion | 1 GB |
| 2³² | | 4 GB |
| 2⁴⁰ | 1 trillion | 1 TB |

*(1 byte = 8 bits; an ASCII character is 1 byte.)*

## 2. Latency numbers every programmer should know

Originally from Jeff Dean (2010 numbers), later visualized by a Google engineer for 2020 hardware. Some numbers have shifted with faster hardware, but the **relative gaps** are the point.

| Operation | Time |
|---|---|
| L1 cache reference | 0.5 ns |
| Branch mispredict | 5 ns |
| L2 cache reference | 7 ns |
| Mutex lock/unlock | 100 ns |
| Main memory reference | 100 ns |
| Compress 1 KB with a fast compressor (Zippy) | 10 µs |
| Send 1 KB over a 1 Gbps network | 10 µs |
| Read 4 KB randomly from SSD | 150 µs |
| Read 1 MB sequentially from memory | 250 µs |
| Round trip within the same data center | 500 µs |
| Read 1 MB sequentially from SSD | 1 ms |
| Disk seek | 10 ms |
| Read 1 MB sequentially from disk | 20 ms |
| Send a packet CA → Netherlands → CA | 150 ms |

*(ns = 10⁻⁹s, µs = 10⁻⁶s = 1,000 ns, ms = 10⁻³s = 1,000 µs)*

**Takeaways:**
- Memory is fast, disk is slow.
- Avoid disk seeks where possible.
- Simple compression algorithms are fast — compress data before sending it over the network if you can.
- Data centers are usually in different regions; sending data between them takes real time.

## 3. Availability numbers

**High availability** = a system stays continuously operational for a long period, measured as a percentage. Most real services land between 99% and 100%.

**SLA (service level agreement)** — the formal contract between a service provider and its customer defining guaranteed uptime. Major cloud providers (AWS, GCP, Azure) commit to 99.9%+ SLAs.

Uptime is talked about in **"nines"** — more nines = less downtime:

| Availability | Downtime / year | Downtime / month | Downtime / day |
|---|---|---|---|
| 99% ("two nines") | ~3.65 days | ~7.2 hours | ~14.4 min |
| 99.9% ("three nines") | ~8.76 hours | ~43.8 min | ~1.44 min |
| 99.99% ("four nines") | ~52.6 min | ~4.38 min | ~8.64 sec |
| 99.999% ("five nines") | ~5.26 min | ~25.9 sec | ~864 ms |

## 4. Worked example — Twitter QPS & storage

**Assumptions** (illustrative, not real Twitter numbers):
- 300 million monthly active users (MAU)
- 50% of users use Twitter daily
- Users post 2 tweets/day on average
- 10% of tweets contain media
- Data retained for 5 years

**QPS estimate:**
1. DAU = 300M × 50% = **150 million**
2. Tweets QPS = 150M × 2 tweets ÷ 24h ÷ 3600s ≈ **~3,500 QPS**
3. Peak QPS = 2 × average QPS ≈ **~7,000 QPS**

**Media storage estimate** (average tweet: `tweet_id` 64 bytes, `text` 140 bytes, `media` 1 MB):
1. Daily media storage = 150M × 2 tweets × 10% × 1 MB = **30 TB/day**
2. 5-year media storage = 30 TB × 365 × 5 ≈ **~55 PB**

## 5. Tips for the interview

- **Round and approximate.** Don't burn time on exact math (`99987 / 9.1` → just do `100,000 / 10`). Precision isn't the point.
- **Write down your assumptions** so you (and the interviewer) can reference them later.
- **Label your units.** "5" is ambiguous — is it 5 KB or 5 MB? Always write "5 MB".
- **Practice the common ones**: QPS, peak QPS, storage, cache size, number of servers needed. These come up repeatedly — rehearse them ahead of time.

## My open questions / follow-ups
- *(add doubts here as they come up)*
