# System Design Part 5: Back-of-the-Envelope Calculations

## Table of Contents
- [Introduction](#introduction)
- [Why It Matters](#why-it-matters)
- [Fundamental Knowledge Required](#fundamental-knowledge-required)
- [Latency Numbers Every Programmer Should Know](#latency-numbers-every-programmer-should-know)
- [Calculation Methodology](#calculation-methodology)
- [Twitter Example: Complete Walkthrough](#twitter-example-complete-walkthrough)
- [Real-World Application](#real-world-application)
- [Best Practices](#best-practices)
- [Common Patterns](#common-patterns)
- [Tools and Resources](#tools-and-resources)

---

## Introduction

**Back-of-the-envelope calculations** are rough estimates used to determine system requirements **before** building the system. The name comes from the practice of doing quick calculations on the back of an envelope - emphasizing that these are approximations, not precise measurements.

### What Are They?

Quick, rough calculations to estimate:
- **QPS** (Queries Per Second)
- **Storage requirements**
- **Bandwidth needs**
- **Memory requirements**
- **Number of servers needed**

### ⚠️ Critical Importance

This is one of the **most overlooked** topics in system design, especially by:
- Fresh graduates
- Junior developers
- Those without production experience

But it's **essential** for:
- Senior engineers
- System architects
- Anyone designing production systems
- Interview success

---

## Why It Matters

### The Real-World Problem

**Scenario: Deploying a New System**

```
DevOps Team: "How many servers do you need?"
You: "Uh... maybe 8 CPU cores?"

DevOps: "Why 8?"
You: "... I don't know, sounds good?"

Result: Either:
❌ Over-provisioned: Wasting $$$
❌ Under-provisioned: System crashes
```

### Questions You'll Face

**From DevOps/Infrastructure Team:**

1. **Compute:**
   - How many servers?
   - CPU cores needed?
   - RAM required?

2. **Database:**
   - Database size?
   - CPU/RAM for database?
   - Read vs write capacity?

3. **Storage:**
   - Disk space needed?
   - SSD or HDD?
   - How much in GB/TB?

4. **Network:**
   - Bandwidth requirements?
   - Expected traffic?

**Without calculations, you're guessing!**

### Consequences of Guessing Wrong

**Over-Provisioning:**
```
Requested: 8 CPU cores, 32GB RAM
Actual usage: 2 CPU cores, 8GB RAM
Waste: 75% of resources
Cost: $500/month vs $125/month needed
Annual waste: $4,500
```

**Under-Provisioning:**
```
Requested: 2 CPU cores, 4GB RAM
Actual need: 4 CPU cores, 16GB RAM
Result: System crashes during peak
Lost revenue: Customers can't access service
Reputation damage: Users lose trust
```

---

## Fundamental Knowledge Required

### 1. Time Conversions

**Seconds in a Day:**
```
24 hours × 60 minutes × 60 seconds = 86,400 seconds

Back-of-envelope approximation:
≈ 100,000 seconds (or 10^5)
≈ 1 lakh seconds (Indian notation)
```

**Rule #1: Always round to make calculations easier**

**Other Time Conversions:**
```
1 minute = 60 seconds
1 hour = 3,600 seconds ≈ 3.6K seconds
1 day = 86,400 seconds ≈ 100K seconds
1 month (30 days) = 2.6M seconds ≈ 2.5M seconds
1 year = 31.5M seconds ≈ 30M seconds
```

### 2. Data Size Units

**Fundamental Conversions:**

```
Bit to Byte:
1 Byte = 8 bits

Bytes to Larger Units (Powers of 1000):
1 KB (Kilobyte)  = 1,000 bytes       = 10^3 bytes
1 MB (Megabyte)  = 1,000 KB          = 10^6 bytes
1 GB (Gigabyte)  = 1,000 MB          = 10^9 bytes
1 TB (Terabyte)  = 1,000 GB          = 10^12 bytes
1 PB (Petabyte)  = 1,000 TB          = 10^15 bytes
```

**Memory Representation:**
```
Pattern: Multiply by 1000 each step

1 byte
   ↓ × 1000
1 KB (thousand bytes)
   ↓ × 1000
1 MB (million bytes)
   ↓ × 1000
1 GB (billion bytes)
   ↓ × 1000
1 TB (trillion bytes)
   ↓ × 1000
1 PB (quadrillion bytes)
```

**Quick Reference:**
```
Text message: ~100 bytes
Small image: ~100 KB
Large image: ~1-5 MB
HD video (1 hour): ~1-5 GB
4K video (1 hour): ~10-50 GB
```

### 3. Approximation Rules

**Golden Rules:**

1. **Round everything**
   ```
   86,400 → 100,000
   99,987 → 100,000
   1,024 → 1,000
   ```

2. **Use powers of 10**
   ```
   1,500 → 1.5K or 2K
   45,000 → 45K or 50K
   ```

3. **Simplify fractions**
   ```
   99,987 / 9.1 → 100,000 / 10 = 10,000
   ```

4. **Accept "good enough"**
   ```
   Don't aim for: 86,432.7 users
   Aim for: ~85K or ~100K users
   ```

---

## Latency Numbers Every Programmer Should Know

### Jeff Dean's Famous List

```
Operation                           Time (Latency)
─────────────────────────────────────────────────
L1 cache reference                  0.5 ns
Branch mispredict                   5 ns
L2 cache reference                  7 ns
Mutex lock/unlock                   100 ns
Main memory reference               100 ns
Compress 1KB with Snappy            10 µs
Send 2KB over 1 Gbps network        20 µs
Read 1 MB sequentially from memory  250 µs
Round trip within same datacenter   500 µs
Disk seek                           10 ms
Read 1 MB sequentially from disk    20 ms
Send packet US → Europe → US        150 ms
```

**Units:**
- ns = nanosecond (10^-9 seconds)
- µs = microsecond (10^-6 seconds)
- ms = millisecond (10^-3 seconds)

### Key Takeaways

**1. Memory is Fast, Disk is Slow**
```
Memory read: 100 ns
Disk read: 10 ms

Disk is 100,000x slower than memory!

This is why:
✅ Redis (in-memory) is fast
❌ Database queries (disk) are slower
```

**2. Avoid Disk Seeks**
```
Disk seek: 10 ms

For 1000 random reads:
1000 × 10 ms = 10 seconds!

Solution: Cache in memory
```

**3. Compression is Cheap**
```
Compress 1KB: 10 µs (very fast)

Benefits:
- Reduce storage costs
- Faster network transfer
- Lower bandwidth usage

Always compress before sending over network!
```

**4. Network Latency Varies by Distance**
```
Same datacenter: 0.5 ms
Cross-continent: 150 ms

300x difference!

Implication:
- Deploy close to users
- Use CDN for global users
- Replicate data across regions
```

---

## Calculation Methodology

### Step-by-Step Process

**1. Gather Requirements**
```
Ask questions:
- Expected users?
- User activity patterns?
- Data per transaction?
- Retention period?
```

**2. Make Assumptions**
```
Write down clearly:
- Monthly Active Users (MAU)
- Daily Active Users (DAU)
- Activity per user per day
- Data size estimates
```

**3. Calculate Key Metrics**
```
Derive:
- Queries per second (QPS)
- Peak QPS
- Storage requirements
- Bandwidth needs
```

**4. Add Safety Margin**
```
Always multiply by 2-3x for:
- Peak traffic handling
- Future growth
- Unexpected spikes
```

---

## Twitter Example: Complete Walkthrough

### Problem Statement

**Estimate Twitter's:**
- QPS (Queries Per Second)
- Storage requirements

### Step 1: Assumptions

Write down assumptions clearly:

```
1. Monthly Active Users (MAU): 300 million
2. Daily Active Users (DAU): 50% of MAU use daily
3. Tweets per user per day: 2 (average)
4. Tweets with media: 10% contain photos/videos
5. Data retention: 5 years minimum
```

**Why these numbers?**
- 300M MAU: Reasonable for a large social network
- 50% DAU: Not everyone uses Twitter daily
- 2 tweets/day: Average (some 0, some 10+)
- 10% with media: Most tweets are text-only
- 5 years: Legal/compliance requirements

### Step 2: Calculate Daily Active Users

```
DAU = MAU × Daily Usage Rate
DAU = 300 million × 50%
DAU = 150 million users/day
```

### Step 3: Calculate Tweets QPS

**Total tweets per day:**
```
Tweets/day = DAU × Tweets per user
Tweets/day = 150 million × 2
Tweets/day = 300 million tweets/day
```

**Tweets per second:**
```
Tweets/second = Tweets/day ÷ Seconds/day
Tweets/second = 300 million ÷ 100,000
Tweets/second ≈ 3,000 tweets/second
```

**Round up:**
```
Average QPS ≈ 3,500 tweets/second
```

**What this means:**
```
Your database must handle:
- 3,500 INSERT operations per second
- Consistently, all day long
```

### Step 4: Calculate Peak QPS

**Peak traffic scenario:**
```
Event: Breaking news (election, disaster, etc.)
Effect: 2x normal traffic

Peak QPS = Average QPS × 2
Peak QPS = 3,500 × 2
Peak QPS ≈ 7,000 tweets/second
```

**System requirement:**
```
Your system MUST handle 7,000 writes/second
Without crashing or slowing down
```

### Step 5: Calculate Storage Requirements

**Data per tweet:**
```
Tweet ID: 64 bytes (UUID)
Tweet text: 140 bytes (average)
Metadata: 32 bytes (timestamp, user ID, etc.)

Total per tweet: ~256 bytes ≈ 250 bytes
```

**Media storage:**
```
Daily tweets with media:
= Total tweets × Media percentage
= 300 million × 10%
= 30 million tweets with media/day

Media size per tweet: 1 MB (average)

Daily media storage:
= 30 million × 1 MB
= 30 million MB
= 30,000 GB
= 30 TB/day
```

**5-year storage:**
```
Days in 5 years = 5 × 365 = 1,825 days

Total storage = 30 TB/day × 1,825 days
Total storage ≈ 55,000 TB
Total storage ≈ 55 PB (Petabytes)
```

### Summary Table

| Metric | Value |
|--------|-------|
| Monthly Active Users | 300 million |
| Daily Active Users | 150 million |
| Average Tweets/Second | 3,500 |
| Peak Tweets/Second | 7,000 |
| Daily Storage (media) | 30 TB |
| 5-Year Storage | 55 PB |

### Database Implications

**Write Capacity:**
```
Must support: 3,500 writes/second (average)
Must support: 7,000 writes/second (peak)

Solution:
- Sharded database
- Write-optimized storage
- SSD for speed
```

**Storage:**
```
Need: 55 PB over 5 years
= 11 PB per year
= ~1 PB per month

Solution:
- Object storage (S3, Blob Storage)
- Tiered storage (hot/cold data)
- Compression
```

---

## Real-World Application

### Personal Experience: Deploying a New System

**Starting from Zero:**

When you have no data, start with **minimum viable resources**:

```
Initial Setup (Dev Environment):
├─ Servers: 1 instance
│  ├─ CPU: 2 cores
│  ├─ RAM: 4 GB
│  └─ Storage: 256 GB SSD
├─ Database: Managed service
│  ├─ CPU: 2 cores
│  └─ RAM: 4 GB
└─ Users: 8-10 (testing team)
```

**Why start small?**
1. **Cost**: Don't waste money
2. **Learning**: Understand actual usage patterns
3. **Optimization**: Find bottlenecks early

### Monitoring and Scaling

**Metrics to Track:**

```
1. CPU Utilization
   - Average: Should be 40-60%
   - Peak: Should not exceed 80%

2. Memory Usage
   - Average: Should be 50-70%
   - Peak: Should not exceed 90%

3. Network I/O
   - Inbound traffic
   - Outbound traffic
   - Identify patterns

4. Database Metrics
   - Queries per second
   - Query latency
   - Connection pool usage

5. Disk I/O
   - Read/write operations
   - Disk queue length
```

**Example Calculation:**

```
Dev Environment Results (10 users):
├─ Avg CPU: 30%
├─ Avg Memory: 2 GB (50%)
├─ Avg QPS: 50
└─ Response time: 200ms

Production Estimate (100 users):
├─ Expected CPU: 30% × 10 = 300% → Need 3-4 cores
├─ Expected Memory: 2GB × 10 = 20GB → Provision 24GB
├─ Expected QPS: 50 × 10 = 500
└─ Safety margin: 2x → Total 8 cores, 48GB RAM
```

### Iterative Approach

```
Phase 1: Dev Environment
├─ Users: 10
├─ Resources: Minimal (2 CPU, 4GB)
└─ Goal: Baseline metrics

Phase 2: Staging
├─ Users: 30 (30% of expected 100)
├─ Resources: 3x dev (6 CPU, 12GB)
└─ Goal: Validate scaling assumptions

Phase 3: Production
├─ Users: 100 (with 2x safety margin)
├─ Resources: 8 CPU, 48GB
└─ Goal: Handle actual load + peaks

Phase 4: Monitor & Adjust
├─ Track actual vs. expected
├─ Adjust based on real data
└─ Plan for growth
```

---

## Best Practices

### 1. Rounding and Approximations

**Always simplify calculations:**

```
❌ Bad: 99,987 / 9.1 = 10,985.93
✅ Good: 100,000 / 10 = 10,000

❌ Bad: 86,432 seconds
✅ Good: ~85K or 100K seconds

❌ Bad: 1,024 bytes per KB
✅ Good: 1,000 bytes per KB
```

**Why?**
- Easier mental math
- Faster calculations
- "Close enough" is acceptable
- Precision doesn't matter in estimates

### 2. Write Down Assumptions

**Always document:**

```
✅ Good Documentation:

Assumptions:
- MAU: 300 million
- DAU: 50% of MAU (150 million)
- Tweets/user/day: 2 (average)
- Media in tweets: 10%
- Retention: 5 years

Calculations:
- Total tweets/day: 300M
- QPS: 3,500
- Peak QPS: 7,000 (2x multiplier)
- Storage/day: 30 TB
- 5-year storage: 55 PB
```

**Why?**
- Reviewable by team
- Can adjust assumptions later
- Shows your reasoning
- Helps in interviews

### 3. Label Your Units

**Always include units:**

```
❌ Bad:
Storage: 5
QPS: 3500
Users: 150

✅ Good:
Storage: 5 TB/day
QPS: 3,500 writes/second
Users: 150 million (DAU)
```

**Common units:**
```
Users: millions, thousands
Time: seconds, milliseconds, hours
Storage: KB, MB, GB, TB, PB
Rate: per second, per day, per month
Percentage: % (always clarify what it's % of)
```

### 4. Common Estimation Patterns

**QPS Calculation:**
```
QPS = (DAU × Actions per user per day) / 100,000

Example:
DAU = 10 million
Actions/user/day = 20

QPS = (10M × 20) / 100K
QPS = 200M / 100K
QPS = 2,000
```

**Peak QPS:**
```
Peak QPS = Average QPS × Multiplier

Multipliers:
- 2x: General purpose
- 3x: Social media (viral content)
- 5x: News/events (breaking news)
- 10x: Flash sales (e-commerce)
```

**Storage Calculation:**
```
Storage = (Records per day) × (Size per record) × (Days retained)

Example:
Records/day = 1 million
Size/record = 1 KB
Retention = 5 years (1,825 days)

Storage = 1M × 1KB × 1,825
Storage = 1.825 billion KB
Storage = 1.825 TB
```

---

## Common Patterns

### Pattern 1: Social Media Platform

```
Given:
- MAU: 500 million
- DAU: 40% of MAU
- Posts/user/day: 1.5
- Media in posts: 20%
- Media size: 2 MB average

Calculate:
1. DAU = 500M × 0.4 = 200M users
2. Posts/day = 200M × 1.5 = 300M posts
3. QPS = 300M / 100K = 3,000
4. Peak QPS = 3,000 × 2 = 6,000
5. Media/day = 300M × 0.2 × 2MB = 120 TB
```

### Pattern 2: Messaging App

```
Given:
- DAU: 100 million
- Messages/user/day: 50
- Message size: 100 bytes
- Media messages: 5%
- Media size: 500 KB

Calculate:
1. Messages/day = 100M × 50 = 5 billion
2. QPS = 5B / 100K = 50,000
3. Text storage/day = 5B × 100 bytes = 500 GB
4. Media/day = 5B × 0.05 × 500KB = 125 TB
```

### Pattern 3: Video Platform

```
Given:
- DAU: 50 million
- Videos watched/user: 10
- Avg video size: 50 MB
- Upload rate: 1% of views

Calculate:
1. Views/day = 50M × 10 = 500M
2. View QPS = 500M / 100K = 5,000
3. Uploads/day = 500M × 0.01 = 5M
4. Storage/day = 5M × 50MB = 250 TB
```

### Pattern 4: E-commerce

```
Given:
- DAU: 10 million
- Orders/day: 5% of DAU
- Order size: 2 KB
- Product views/user: 20

Calculate:
1. Orders/day = 10M × 0.05 = 500K
2. Order QPS = 500K / 100K = 5
3. Views/day = 10M × 20 = 200M
4. View QPS = 200M / 100K = 2,000
5. Read-heavy: 2000:5 (400:1 ratio)
```

---

## Practical Examples

### Example 1: Image Sharing App

**Requirements:**
Design storage for Instagram-like app

**Assumptions:**
```
- DAU: 100 million
- Photos uploaded/user/day: 2
- Photo size: 2 MB average
- Retention: Forever (10 years estimate)
```

**Calculations:**
```
Photos/day = 100M × 2 = 200M photos
Storage/day = 200M × 2MB = 400,000 MB = 400 GB
Wait, that seems too low!

Actually:
400,000 MB = 400,000,000 KB
= 400 GB? No!
= 200M photos × 2 MB = 400 TB/day ✓

10-year storage = 400 TB × 365 × 10
= 1.46 million TB
= 1,460 PB
≈ 1.5 Exabytes
```

### Example 2: URL Shortener

**Requirements:**
Design bit.ly-like service

**Assumptions:**
```
- New URLs/month: 100 million
- Read-write ratio: 100:1
- URL storage: 500 bytes
- Retention: 5 years
```

**Calculations:**
```
Writes:
- URLs/month = 100M
- URLs/second = 100M / (30 × 24 × 3600)
= 100M / 2.5M
≈ 40 writes/second

Reads:
- Read/Write ratio = 100:1
- Read QPS = 40 × 100 = 4,000

Storage:
- Total URLs in 5 years = 100M × 12 × 5 = 6 billion
- Storage = 6B × 500 bytes = 3 TB
```

### Example 3: Notification System

**Requirements:**
Push notification service

**Assumptions:**
```
- Registered devices: 500 million
- Active devices: 30%
- Notifications/device/day: 10
- Notification size: 1 KB
```

**Calculations:**
```
Active devices = 500M × 0.3 = 150M
Notifications/day = 150M × 10 = 1.5 billion
QPS = 1.5B / 100K = 15,000
Peak QPS = 15K × 3 = 45,000 (breaking news)

Storage:
- If storing 30 days: 1.5B × 30 × 1KB = 45 TB
```

---

## Advanced Considerations

### 1. Read vs Write Ratio

**Impact on architecture:**

```
Write-Heavy (1:1 ratio):
- Social media posts
- Chat messages
Solution: Write-optimized DB, sharding

Read-Heavy (100:1 ratio):
- News sites
- Product catalogs
Solution: Heavy caching, CDN, read replicas

Balanced (10:1 ratio):
- E-commerce
- Dashboards
Solution: Cache + optimized DB queries
```

### 2. Geographic Distribution

**Latency impact:**

```
Single Region:
- Latency: 20-100ms
- Simple architecture
- Lower cost

Multi-Region:
- Latency: Reduced for users
- Complex replication
- Higher cost

Global:
- CDN essential
- Data residency concerns
- 3-5x infrastructure cost
```

### 3. Growth Projections

**Plan for scale:**

```
Year 1: 1 million users
Year 2: 5 million (5x growth)
Year 3: 20 million (4x growth)
Year 4: 50 million (2.5x growth)
Year 5: 100 million (2x growth)

Implication:
- Start with 1M capacity
- Design for 100M scale
- Auto-scaling crucial
```

---

## Tools and Resources

### Calculation Tools

**Online Calculators:**
- [System Design Calculator](https://bytebytego.com/calculator)
- AWS Pricing Calculator
- GCP Pricing Calculator
- Azure Calculator

### Reference Charts

**Quick Reference Card:**
```
Time:
- 1 day ≈ 100K seconds
- 1 month ≈ 2.5M seconds
- 1 year ≈ 30M seconds

Storage:
- 1 KB = 1,000 bytes
- 1 MB = 1,000 KB
- 1 GB = 1,000 MB
- 1 TB = 1,000 GB

QPS Ranges:
- Small: < 100
- Medium: 100-10K
- Large: 10K-100K
- Huge: > 100K
```

### Learning Resources

**Articles:**
- Jeff Dean's Latency Numbers
- ByteByteGo System Design Primer
- High Scalability Blog

**Books:**
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "System Design Interview" by Alex Xu

---

## Interview Tips

### How to Approach in Interviews

**Step 1: Clarify Requirements**
```
Ask:
- How many users?
- Read vs write ratio?
- Data retention?
- Geographic distribution?
```

**Step 2: State Assumptions**
```
Say:
"I'm assuming:
- 100 million DAU
- 50% are power users
- Average 10 actions/day
- Does this sound reasonable?"
```

**Step 3: Calculate Out Loud**
```
"Let me calculate QPS:
100M users × 10 actions = 1 billion actions/day
1 billion / 100K seconds = 10K QPS
Adding 2x for peak: 20K QPS"
```

**Step 4: Validate**
```
Ask:
"Does this QPS seem reasonable?
Should I account for any spikes?"
```

### Common Mistakes to Avoid

```
❌ Being too precise (86,432.7 users)
✅ Round numbers (85K or 100K users)

❌ Not stating assumptions
✅ Clearly document all assumptions

❌ Forgetting units
✅ Always label (MB, seconds, etc.)

❌ Ignoring peak traffic
✅ Calculate peak QPS (2-3x average)

❌ No safety margin
✅ Add buffer for growth
```

---

## Key Takeaways

### Core Principles

1. **It's About Approximations**
   - Not exact calculations
   - "Good enough" is perfect
   - Round everything

2. **Document Everything**
   - Write assumptions
   - Show calculations
   - Label units

3. **Learn from Experience**
   - Monitor actual usage
   - Compare to estimates
   - Adjust next time

4. **Start Small, Scale Up**
   - Begin with minimum
   - Monitor metrics
   - Scale based on data

### Essential Formulas

```
QPS = (DAU × Actions per day) / 100,000

Peak QPS = Average QPS × 2 (or 3, or 5)

Storage = Records × Size × Retention Days

Bandwidth = QPS × Average Request Size
```

### Remember

> "Back-of-the-envelope calculations are not about precision. They're about having a reasonable starting point and understanding the scale of your system. Experience makes you better at this - every system you build teaches you more about realistic estimates."

---

## Conclusion

Back-of-the-envelope calculations are:
- **Essential** for system design
- **Simple** once you practice
- **Valuable** in interviews and real work
- **Learnable** through experience

**Start practicing with:**
- Twitter clone
- Instagram clone
- WhatsApp clone
- YouTube clone

The more you practice, the more intuitive these calculations become!

---

**Version**: 5.0  
**Last Updated**: 2024  
**Part**: 5 of System Design Series  
**Topic**: Back-of-the-Envelope Calculations  
**Origin**: Jeff Dean, Google  

---

*"The best engineers don't guess. They estimate. And they get better at estimating with every system they build."*

Happy Learning! 🚀