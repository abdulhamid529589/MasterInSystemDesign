# System Design Part 6: Rate Limiting

## Table of Contents
- [Introduction](#introduction)
- [What is Rate Limiting?](#what-is-rate-limiting)
- [Why Rate Limiting Matters](#why-rate-limiting-matters)
- [Rate Limiting Algorithms](#rate-limiting-algorithms)
  - [Token Bucket Algorithm](#token-bucket-algorithm)
  - [Leaky Bucket Algorithm](#leaky-bucket-algorithm)
  - [Fixed Window Counter Algorithm](#fixed-window-counter-algorithm)
  - [Sliding Window Log Algorithm](#sliding-window-log-algorithm)
  - [Sliding Window Counter Algorithm](#sliding-window-counter-algorithm)
- [Algorithm Comparison](#algorithm-comparison)
- [Implementation Strategies](#implementation-strategies)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)

---

## Introduction

**Rate Limiting** is a critical system design pattern that controls the rate of traffic sent by a client or service. It's one of the most important topics in system design, yet often overlooked.

### What You'll Learn

- What rate limiting is and why it's essential
- 5 different rate limiting algorithms
- Trade-offs of each approach
- How companies like Amazon and Stripe use rate limiting
- Implementation strategies

---

## What is Rate Limiting?

### The Basic Concept

```
Traditional Flow (No Rate Limiting):
User → Server → Process → Response

Problem:
- Unlimited requests
- Server overwhelm
- System crash

With Rate Limiting:
User → Rate Limiter → Server → Process → Response
              ↓
         (Check limits)
              ↓
         (429 if exceeded)
```

### The Problem

**Scenario:**
```
Server Capacity: 100 requests/minute
Incoming Traffic: 150 requests/minute

Without Rate Limiting:
→ Server overloaded
→ Performance degradation
→ Risk of crash

With Rate Limiting:
→ Accept 100 requests/minute
→ Reject 50 requests/minute (429 error)
→ Server stays healthy
```

### HTTP 429 Error

**Too Many Requests:**
```
HTTP Status Code: 429
Message: "Too Many Requests"
Meaning: Client has sent too many requests

Headers often include:
- X-RateLimit-Limit: Maximum allowed
- X-RateLimit-Remaining: Requests left
- X-RateLimit-Reset: Time until reset
- Retry-After: Seconds to wait
```

---

## Why Rate Limiting Matters

### 1. Prevent Resource Exhaustion

**Without Rate Limiting:**
```
Scenario: Single user sends 10,000 requests/second

Impact:
- CPU: 100% usage
- Memory: Exhausted
- Database: Connection pool depleted
- Other users: Can't access service
- Result: System crash
```

**With Rate Limiting:**
```
Policy: 100 requests/minute per user

Result:
- Excess requests rejected
- Server resources protected
- All users get fair access
- System stays stable
```

### 2. Prevent DoS Attacks

**Attack Scenario:**
```
Malicious Actor:
- Sends 1 million requests/second
- Attempts to overwhelm system
- Locks out legitimate users

Rate Limiter Response:
- Detects abnormal pattern
- Blocks excess requests
- Protects infrastructure
- Maintains availability
```

### 3. Cost Control

**Cloud Infrastructure Costs:**
```
Without Rate Limiting:
- Attacker sends 10M requests
- Auto-scaling triggers
- Spins up 100 servers
- Monthly bill: $50,000

With Rate Limiting:
- Blocks after 1,000 requests
- No excessive scaling
- Monthly bill: $500
- Savings: $49,500
```

### 4. Fair Usage

**Shared Service:**
```
SaaS Platform with 1,000 users

Without Rate Limiting:
- Power user: 90% of resources
- Other 999 users: Fight for 10%
- Poor experience for majority

With Rate Limiting:
- Each user: Fair allocation
- No single user dominates
- Consistent performance for all
```

---

## Rate Limiting Algorithms

### 1. Token Bucket Algorithm

**Most widely used** by companies like Amazon and Stripe.

#### How It Works

```
Components:
┌─────────────────┐
│  Token Bucket   │
│                 │
│  Capacity: 5    │
│  Tokens: ●●●    │
└─────────────────┘
       ↑
       │
   Refiller
   (3 tokens/sec)
```

**Algorithm:**

1. **Bucket** holds tokens (max capacity)
2. **Refiller** adds tokens at fixed rate
3. **Request arrives** → Take 1 token
4. **Token available?** → Process request
5. **No tokens?** → Reject (429 error)

#### Step-by-Step Example

**Configuration:**
```
Bucket Capacity: 5 tokens
Refill Rate: 3 tokens/second
```

**Timeline:**

```
Second 1:
├─ Refiller adds 3 tokens → Bucket: ●●●
├─ Request 1 arrives → Takes 1 token → Bucket: ●●
├─ Request 2 arrives → Takes 1 token → Bucket: ●
├─ Request 3 arrives → Takes 1 token → Bucket: (empty)
└─ Result: 3 requests processed

Second 2:
├─ Refiller adds 3 tokens → Bucket: ●●●
├─ Request 4 arrives → Takes 1 token → Bucket: ●●
├─ Request 5 arrives → Takes 1 token → Bucket: ●
├─ Request 6 arrives → Takes 1 token → Bucket: (empty)
├─ Request 7 arrives → NO TOKENS → REJECTED (429)
└─ Result: 3 processed, 1 rejected

Second 3:
├─ Refiller adds 3 tokens → Bucket: ●●●
├─ No requests
└─ Bucket: ●●●

Second 4:
├─ Refiller tries to add 3 tokens
├─ Bucket capacity: 5 (already has 3)
├─ Only 2 tokens added → Bucket: ●●●●● (full)
└─ Excess tokens discarded
```

#### Code Example

```javascript
class TokenBucket {
  constructor(capacity, refillRate) {
    this.capacity = capacity;
    this.tokens = 0;
    this.refillRate = refillRate; // tokens per second
    this.lastRefill = Date.now();
  }
  
  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000; // seconds
    const tokensToAdd = Math.floor(elapsed * this.refillRate);
    
    this.tokens = Math.min(
      this.capacity, 
      this.tokens + tokensToAdd
    );
    this.lastRefill = now;
  }
  
  allowRequest() {
    this.refill();
    
    if (this.tokens > 0) {
      this.tokens--;
      return true; // Allow request
    }
    
    return false; // Reject request (429)
  }
}

// Usage
const limiter = new TokenBucket(5, 3); // 5 capacity, 3/sec

if (limiter.allowRequest()) {
  processRequest();
} else {
  return429Error();
}
```

#### Pros and Cons

**Pros:**
```
✅ Easy to implement
✅ Memory efficient
✅ Allows burst traffic
   - Bucket can accumulate tokens
   - Short spikes handled gracefully
✅ Well understood
✅ Used by major companies
```

**Cons:**
```
❌ Two parameters to tune:
   1. Bucket capacity
   2. Refill rate
   
❌ Difficult to tune properly
   - Too small: Legitimate bursts rejected
   - Too large: Ineffective protection
   
❌ Requires experimentation
```

---

### 2. Leaky Bucket Algorithm

Works like a **queue system** with fixed processing rate.

#### How It Works

```
Analogy: Bucket with a hole

┌─────────────────────┐
│   Incoming           │
│   Requests           │
│   (fast rate)        │
│      ↓↓↓↓↓           │
├─────────────────────┤
│                     │
│   ████████████      │ ← Bucket
│   ████████████      │   (Queue)
│   ████████████      │
│                     │
└──────────┬──────────┘
           │ (small hole)
           ↓
      Fixed Rate
   (5 requests/min)
```

**Concept:**
- Requests fill bucket (like water)
- Requests leak out at fixed rate (small hole)
- Bucket full → Reject new requests

#### Real-World Analogy

**Shower Example:**
```
Overhead Tank: 1,500 liters (many requests)
Direct Connection: Too much pressure!

Solution: Shower head
- Limits water flow
- Comfortable rate
- Prevents overwhelm

Same concept:
- Requests pour in fast
- Leak out slowly (controlled)
- Server handles comfortably
```

#### Step-by-Step Example

**Configuration:**
```
Bucket Capacity: 10,000 requests
Outflow Rate: 5 requests/minute
```

**Timeline:**

```
Minute 1:
├─ 20,000 requests arrive
├─ Bucket capacity: 10,000
├─ Accepted: 10,000 (queued)
├─ Rejected: 10,000 (overflow)
├─ Processing: 5 requests/minute
└─ Bucket: 9,995 remaining

Minute 2:
├─ No new requests
├─ Processing: 5 requests/minute
└─ Bucket: 9,990 remaining

Minute 3:
├─ 100 requests arrive
├─ Added to queue
├─ Processing: 5 requests/minute
└─ Bucket: 10,085 remaining
```

#### Code Example

```javascript
class LeakyBucket {
  constructor(capacity, outflowRate) {
    this.capacity = capacity;
    this.queue = [];
    this.outflowRate = outflowRate; // requests per minute
    this.lastLeak = Date.now();
  }
  
  leak() {
    const now = Date.now();
    const elapsed = (now - this.lastLeak) / 60000; // minutes
    const leakCount = Math.floor(elapsed * this.outflowRate);
    
    // Process (remove) leaked requests
    this.queue.splice(0, leakCount);
    this.lastLeak = now;
  }
  
  addRequest(request) {
    this.leak();
    
    // Check if bucket is full
    if (this.queue.length >= this.capacity) {
      return false; // Reject (429)
    }
    
    // Add to queue
    this.queue.push(request);
    return true; // Accepted
  }
  
  processRequests() {
    // Background worker processes queue at fixed rate
    setInterval(() => {
      if (this.queue.length > 0) {
        const request = this.queue.shift();
        handleRequest(request);
      }
    }, 60000 / this.outflowRate); // Rate in milliseconds
  }
}

// Usage
const limiter = new LeakyBucket(10000, 5);

if (limiter.addRequest(req)) {
  console.log('Request queued');
} else {
  return429Error();
}
```

#### Pros and Cons

**Pros:**
```
✅ Memory efficient (given limited queue)
✅ Fixed processing rate
   - Stable output
   - Predictable load
   
✅ Good for steady workloads
✅ Smooths traffic spikes
```

**Cons:**
```
❌ Burst traffic fills queue with old requests
   - New requests can't get through
   - Stale requests processed first
   
❌ Two parameters to tune:
   1. Bucket capacity
   2. Leak rate (outflow)
   
❌ FIFO may not be desired
   - Old requests processed first
   - New requests wait
```

---

### 3. Fixed Window Counter Algorithm

Divides timeline into **fixed time windows** and counts requests.

#### How It Works

```
Timeline divided into fixed windows:

Second 1    Second 2    Second 3
├─────────┼─────────┼─────────┤
│   ●●●   │   ●●    │   ●●●   │
│  Count:3│  Count:2│  Count:3│
└─────────┴─────────┴─────────┘

Policy: 3 requests/second

Second 1: 3 requests → OK
Second 2: 2 requests → OK  
Second 3: 3 requests → OK, 4th rejected
```

#### Step-by-Step Example

**Configuration:**
```
Window Size: 1 second
Threshold: 3 requests/second
```

**Timeline:**

```
Time: 0.0 - 1.0 seconds (Window 1)
├─ Request 1 (0.1s) → Counter: 1 → Allow
├─ Request 2 (0.5s) → Counter: 2 → Allow
├─ Request 3 (0.9s) → Counter: 3 → Allow
└─ Window ends → Reset counter

Time: 1.0 - 2.0 seconds (Window 2)
├─ Counter reset to 0
├─ Request 4 (1.1s) → Counter: 1 → Allow
├─ Request 5 (1.1s) → Counter: 2 → Allow
├─ Request 6 (1.2s) → Counter: 3 → Allow
├─ Request 7 (1.5s) → Counter: 4 → REJECT (429)
└─ Window ends → Reset counter

Time: 2.0 - 3.0 seconds (Window 3)
├─ Counter reset to 0
├─ Request 8 (2.3s) → Counter: 1 → Allow
└─ ...
```

#### The Edge Problem

**Critical Bug:**

```
Problem Scenario:

Window 1: [0.0 - 1.0]     Window 2: [1.0 - 2.0]
         ├─────────┼─────────┤
         │     ●●  │  ●●     │
         │  (0.8s) │ (1.2s)  │
         └─────────┴─────────┘

Highlighted: [0.8 - 1.2] = 0.4 seconds
In this 0.4 second window:
- 2 requests at 0.8-0.9s (Window 1)
- 2 requests at 1.0-1.2s (Window 2)
- Total: 4 requests in 0.4 seconds!

But policy is 3 requests/second!
```

**The Issue:**
```
Policy: 3 requests/second maximum

Windows see:
- Window 1: 2 requests → OK ✓
- Window 2: 2 requests → OK ✓

Reality:
- Actual window (0.8-1.2s): 4 requests
- Exceeds limit by 33%!
- System vulnerability
```

#### Code Example

```javascript
class FixedWindowCounter {
  constructor(windowSize, threshold) {
    this.windowSize = windowSize; // seconds
    this.threshold = threshold;
    this.counter = 0;
    this.windowStart = Date.now();
  }
  
  allowRequest() {
    const now = Date.now();
    const elapsed = (now - this.windowStart) / 1000;
    
    // Check if window expired
    if (elapsed >= this.windowSize) {
      // Reset counter for new window
      this.counter = 0;
      this.windowStart = now;
    }
    
    // Check threshold
    if (this.counter < this.threshold) {
      this.counter++;
      return true; // Allow
    }
    
    return false; // Reject (429)
  }
}

// Usage
const limiter = new FixedWindowCounter(1, 3); // 1 second, 3 requests

if (limiter.allowRequest()) {
  processRequest();
} else {
  return429Error();
}
```

#### Pros and Cons

**Pros:**
```
✅ Very easy to implement
✅ Memory efficient
✅ Resets quota at window boundary
   - Clear accounting
   - Easy to understand
```

**Cons:**
```
❌ Edge problem (burst at boundaries)
   - Can exceed limits
   - System vulnerability
   
❌ Not suitable for strict limits
❌ Spike at window edges
```

---

### 4. Sliding Window Log Algorithm

Maintains a **log of timestamps** to solve the edge problem.

#### How It Works

```
Concept: Track every request timestamp

Request Log:
┌──────────────────────────────┐
│ Timestamps:                  │
│ [1.0, 3.0, 4.5, 6.0, 8.0]   │
└──────────────────────────────┘

Current time: 8.5 seconds
Window size: 5 seconds
Window: [3.5 - 8.5]

Check log:
- 1.0 → Outside window (remove)
- 3.0 → Outside window (remove)  
- 4.5 → Inside window ✓
- 6.0 → Inside window ✓
- 8.0 → Inside window ✓

Active requests: 3
```

#### Step-by-Step Example

**Configuration:**
```
Window Size: 5 seconds
Threshold: 3 requests per 5 seconds
```

**Timeline:**

```
Time: 1.0s
├─ Request arrives
├─ Log: [1.0]
├─ Count: 1 ≤ 3 → Allow
└─ Log: [1.0]

Time: 3.0s
├─ Request arrives
├─ Window: [0.0 - 3.0] (3s back from now)
├─ Active: [1.0, 3.0]
├─ Count: 2 ≤ 3 → Allow
└─ Log: [1.0, 3.0]

Time: 4.0s
├─ Request arrives
├─ Window: [0.0 - 4.0]
├─ Active: [1.0, 3.0, 4.0]
├─ Count: 3 ≤ 3 → Allow
└─ Log: [1.0, 3.0, 4.0]

Time: 6.0s
├─ Request arrives
├─ Window: [1.0 - 6.0] (5s back from now)
├─ Remove expired: 1.0 is >5s old
├─ Active: [3.0, 4.0, 6.0]
├─ Count: 3 ≤ 3 → Allow
└─ Log: [3.0, 4.0, 6.0]

Time: 7.0s
├─ Request arrives
├─ Window: [2.0 - 7.0]
├─ Remove expired: None
├─ Active: [3.0, 4.0, 6.0, 7.0]
├─ Count: 4 > 3 → REJECT (429)
└─ Log: [3.0, 4.0, 6.0] (not added)

Time: 8.0s
├─ Request arrives
├─ Window: [3.0 - 8.0]
├─ Remove expired: None
├─ Active: [3.0, 4.0, 6.0]
├─ Count: 3 ≤ 3 → Allow
└─ Log: [3.0, 4.0, 6.0, 8.0]
```

#### Code Example

```javascript
class SlidingWindowLog {
  constructor(windowSize, threshold) {
    this.windowSize = windowSize; // seconds
    this.threshold = threshold;
    this.log = []; // Array of timestamps
  }
  
  cleanLog(now) {
    // Remove timestamps outside window
    const cutoff = now - (this.windowSize * 1000);
    this.log = this.log.filter(timestamp => timestamp > cutoff);
  }
  
  allowRequest() {
    const now = Date.now();
    
    // Clean expired entries
    this.cleanLog(now);
    
    // Check threshold
    if (this.log.length < this.threshold) {
      // Add current request timestamp
      this.log.push(now);
      return true; // Allow
    }
    
    return false; // Reject (429)
  }
}

// Usage with cache (Redis)
class DistributedSlidingWindowLog {
  constructor(redis, windowSize, threshold) {
    this.redis = redis;
    this.windowSize = windowSize;
    this.threshold = threshold;
  }
  
  async allowRequest(userId) {
    const now = Date.now();
    const key = `rate_limit:${userId}`;
    const cutoff = now - (this.windowSize * 1000);
    
    // Remove expired entries
    await this.redis.zremrangebyscore(key, 0, cutoff);
    
    // Count current requests
    const count = await this.redis.zcard(key);
    
    if (count < this.threshold) {
      // Add current timestamp
      await this.redis.zadd(key, now, now);
      await this.redis.expire(key, this.windowSize);
      return true;
    }
    
    return false;
  }
}
```

#### Pros and Cons

**Pros:**
```
✅ Solves edge problem
   - Accurate at any time
   - No boundary issues
   
✅ Precise rate limiting
✅ Handles any time window
```

**Cons:**
```
❌ Memory intensive
   - Stores every timestamp
   - High traffic = large log
   - Example: 10K req/s = 10K timestamps
   
❌ Performance overhead
   - Must scan/clean log
   - Filter expired entries
   - O(n) operation
   
❌ Not scalable for very high traffic
```

---

### 5. Sliding Window Counter Algorithm

**Hybrid approach** combining Fixed Window Counter and Sliding Window Log.

#### How It Works

```
Combines:
- Fixed windows (memory efficient)
- Sliding window (accurate)

Formula:
Current Rate = 
  (Previous Window × Overlap %) + 
  (Current Window Count)
```

#### The Clever Math

**Example:**

```
Configuration:
- Window size: 1 second
- Threshold: 3 requests/second

Timeline:
Window 1: [3.0-4.0s]    Window 2: [4.0-5.0s]
         ├─────────┼─────────┤
         │   ●●●   │   ●     │
         │ Count:3 │ Count:1 │
         └─────────┴─────────┘

Request arrives at: 4.3 seconds

Calculate:
1. How much of previous window is relevant?
   - Current time: 4.3s
   - Window size: 1s
   - Relevant from: 3.3s
   - Previous window: 3.0-4.0s
   - Overlap: 70% of previous window (3.3-4.0)

2. Weighted calculation:
   Previous contribution: 3 × 0.7 = 2.1
   Current contribution: 1
   Total: 2.1 + 1 = 3.1
   
3. Decision:
   3.1 > 3.0 → REJECT
```

**Another Example:**

```
Request at: 4.1 seconds

Calculate:
1. Overlap with previous window:
   - Relevant from: 3.1s
   - Overlap: 90% (3.1-4.0 is 90% of 3.0-4.0)

2. Weighted:
   Previous: 3 × 0.9 = 2.7
   Current: 1
   Total: 3.7 > 3.0 → REJECT
```

#### Code Example

```javascript
class SlidingWindowCounter {
  constructor(windowSize, threshold) {
    this.windowSize = windowSize; // seconds
    this.threshold = threshold;
    this.windows = new Map(); // timestamp -> count
  }
  
  getWindowKey(timestamp) {
    // Round down to window boundary
    return Math.floor(timestamp / this.windowSize) * this.windowSize;
  }
  
  allowRequest() {
    const now = Date.now() / 1000; // Convert to seconds
    const currentWindowKey = this.getWindowKey(now);
    const previousWindowKey = currentWindowKey - this.windowSize;
    
    // Get counts
    const currentCount = this.windows.get(currentWindowKey) || 0;
    const previousCount = this.windows.get(previousWindowKey) || 0;
    
    // Calculate overlap with previous window
    const elapsedInCurrent = now - currentWindowKey;
    const overlapPercentage = 1 - (elapsedInCurrent / this.windowSize);
    
    // Weighted count
    const weightedCount = 
      (previousCount * overlapPercentage) + currentCount;
    
    if (weightedCount < this.threshold) {
      // Increment current window
      this.windows.set(currentWindowKey, currentCount + 1);
      
      // Clean old windows
      this.windows.delete(previousWindowKey - this.windowSize);
      
      return true; // Allow
    }
    
    return false; // Reject (429)
  }
}

// Usage
const limiter = new SlidingWindowCounter(1, 3); // 1s window, 3 req/s

if (limiter.allowRequest()) {
  processRequest();
} else {
  return429Error();
}
```

#### Pros and Cons

**Pros:**
```
✅ Solves edge problem (mostly)
✅ Memory efficient
   - Only stores window counts
   - Not every timestamp
   
✅ Good balance of accuracy and efficiency
✅ Used in production systems
```

**Cons:**
```
❌ Approximate (not 100% accurate)
   - Uses weighted average
   - Close enough for most cases
   
❌ Slight edge cases remain
❌ More complex than simple counters
```

---

## Algorithm Comparison

### Feature Comparison

| Algorithm | Memory | Accuracy | Burst Handling | Complexity | Production Use |
|-----------|--------|----------|----------------|------------|----------------|
| **Token Bucket** | Low | Good | Excellent | Low | ✅ Amazon, Stripe |
| **Leaky Bucket** | Low | Good | Poor (queues fill) | Medium | ✅ Traffic shaping |
| **Fixed Window** | Very Low | Poor (edges) | Medium | Very Low | ⚠️ Simple systems |
| **Sliding Log** | High | Excellent | Good | High | ⚠️ Low traffic only |
| **Sliding Counter** | Low | Very Good | Good | Medium | ✅ Many systems |

### When to Use Each

**Token Bucket:**
```
✅ API rate limiting
✅ Need burst tolerance
✅ Variable traffic patterns
✅ User-facing APIs

Examples:
- AWS API Gateway
- Stripe API
- REST APIs
```

**Leaky Bucket:**
```
✅ Traffic shaping
✅ Need stable output
✅ Backend processing
✅ Message queues

Examples:
- Network routers
- Video streaming
- Background jobs
```

**Fixed Window Counter:**
```
✅ Simple use cases
✅ Low traffic
✅ Approximation OK
✅ Quick implementation

Examples:
- Internal tools
- Admin dashboards
- Development/staging
```

**Sliding Window Log:**
```
✅ Strict accuracy required
✅ Low-medium traffic
✅ Regulatory compliance
✅ Financial systems

Examples:
- Payment processing
- Stock trading APIs
- Healthcare systems
```

**Sliding Window Counter:**
```
✅ High traffic systems
✅ Need accuracy
✅ Limited memory
✅ Production systems

Examples:
- Social media platforms
- E-commerce sites
- Large-scale APIs
```

---

## Implementation Strategies

### 1. Where to Implement

**API Gateway Level:**
```
Pros:
- Centralized control
- Protects all services
- Single point of management

Cons:
- Single point of failure
- Can't customize per service

Example: AWS API Gateway, Kong, Nginx
```

**Application Level:**
```
Pros:
- Fine-grained control
- Service-specific policies
- More flexible

Cons:
- Must implement in each service
- Harder to manage

Example: Express middleware, Django middleware
```

**Proxy/Load Balancer:**
```
Pros:
- Protects infrastructure
- Handles distributed load
- Transparent to application

Cons:
- Limited context
- Generic policies only

Example: Nginx, HAProxy, Envoy
```

### 2. Storage Options

**In-Memory (Local):**
```javascript
// Simple, single server
const counters = new Map();

function rateLimit(userId) {
  const count = counters.get(userId) || 0;
  if (count >= limit) return false;
  counters.set(userId, count + 1);
  return true;
}

Pros: Fast, simple
Cons: Not distributed, lost on restart
```

**Redis (Distributed):**
```javascript
// Distributed, persistent
async function rateLimit(userId) {
  const key = `rate:${userId}`;
  const count = await redis.incr(key);
  
  if (count === 1) {
    await redis.expire(key, window);
  }
  
  return count <= limit;
}

Pros: Distributed, persistent, atomic
Cons: Network latency, external dependency
```

**Database:**
```sql
-- Persistent, queryable
CREATE TABLE rate_limits (
  user_id VARCHAR(255),
  window_start TIMESTAMP,
  request_count INT,
  PRIMARY KEY (user_id, window_start)
);

Pros: Persistent, auditable
Cons: Slow, database load
```

### 3. Rate Limit Dimensions

**Per User:**
```
Rate limit by user ID
- 100 requests/minute per user
- Fair usage across users
```

**Per IP Address:**
```
Rate limit by source IP
- Protects against DDoS
- No authentication needed
```

**Per API Key:**
```
Rate limit by API key
- Different tiers (free/paid)
- Easy to track/revoke
```

**Per Endpoint:**
```
Rate limit by route
- /search: 10/min (expensive)
- /health: unlimited (cheap)
```

**Combined:**
```
Hierarchical limits:
- Global: 10,000 req/s
- Per user: 100 req/min
- Per endpoint: 10 req/min
```

---

## Real-World Examples

### Example 1: AWS API Gateway

**Implementation:**
```
Algorithm: Token Bucket

Limits:
- Steady-state: 10,000 req/s
- Burst: 5,000 requests

How it works:
- Bucket refills at 10,000 tokens/s
- Bucket capacity: 5,000 tokens
- Allows burst up to 5,000
- Then steady at 10,000 req/s
```

### Example 2: GitHub API

**Implementation:**
```
Algorithm: Sliding Window (hourly)

Limits:
- Authenticated: 5,000 req/hour
- Unauthenticated: 60 req/hour

Headers:
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1640995200
X-RateLimit-Used: 1
```

### Example 3: Twitter API

**Implementation:**
```
Algorithm: Fixed Window (15 min)

Limits (per 15-min window):
- Tweet timeline: 900 requests
- User lookup: 900 requests
- Tweet search: 180 requests
- Post tweet: 300 requests

Response on exceed:
{
  "errors": [{
    "code": 88,
    "message": "Rate limit exceeded"
  }]
}
```

### Example 4: Stripe API

**Implementation:**
```
Algorithm: Token Bucket

Default limits:
- 100 read requests/second
- 100 write requests/second

Features:
- Different limits for live vs test
- Automatic retry with exponential backoff
- Rate limit headers in response
```

---

## Best Practices

### 1. Communicate Limits Clearly

**Include in Response Headers:**
```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 99
X-RateLimit-Reset: 1640995200
Content-Type: application/json

{
  "data": "..."
}
```

**On Rate Limit Exceeded:**
```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640995200
Retry-After: 60
Content-Type: application/json

{
  "error": "Rate limit exceeded",
  "message": "You have exceeded 100 requests per minute",
  "retryAfter": 60
}
```

### 2. Implement Retry Logic

**Client-Side Retry:**
```javascript
async function apiRequestWithRetry(url, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    const response = await fetch(url);
    
    if (response.status === 429) {
      const retryAfter = response.headers.get('Retry-After');
      const delay = retryAfter ? parseInt(retryAfter) * 1000 : 
                    Math.pow(2, i) * 1000; // Exponential backoff
      
      console.log(`Rate limited. Retrying after ${delay}ms`);
      await sleep(delay);
      continue;
    }
    
    return response;
  }
  
  throw new Error('Max retries exceeded');
}
```

### 3. Different Tiers

**Tiered Limits:**
```
Free Tier:
- 100 requests/hour
- Basic endpoints only

Pro Tier:
- 10,000 requests/hour
- All endpoints
- Higher burst

Enterprise:
- 100,000 requests/hour
- All endpoints
- Dedicated support
- Custom limits
```

### 4. Gradual Degradation

**Progressive Limits:**
```
Instead of hard block at limit:

80% of limit: Warning header
X-RateLimit-Warning: "Approaching limit"

90% of limit: Slow down responses
- Add 100ms delay
- Encourage backoff

100% of limit: Block
- Return 429
- Provide retry guidance
```

### 5. Monitoring and Alerting

**Key Metrics:**
```
Monitor:
- Rate limit hits per user
- Rate limit hits per endpoint
- 429 error rate
- Average requests per user

Alert on:
- Sudden spike in 429s
- User consistently hitting limits
- Potential abuse patterns
- System-wide rate limit failures
```

### 6. Whitelist/Blacklist

**Exception Handling:**
```
Whitelist:
- Internal services (unlimited)
- Trusted partners (higher limits)
- Paid tier users (custom limits)

Blacklist:
- Known attackers (block completely)
- Abusive users (very low limits)
- Suspicious patterns (temporary block)
```

---

## Key Takeaways

### Core Concepts

1. **Rate Limiting is Essential**
   - Protects server resources
   - Prevents abuse
   - Ensures fair usage
   - Controls costs

2. **Choose Algorithm Based on Needs**
   - Token Bucket: Most versatile
   - Leaky Bucket: Stable output
   - Fixed Window: Simplest
   - Sliding Log: Most accurate
   - Sliding Counter: Best balance

3. **Implementation Matters**
   - Where: Gateway, app, or proxy
   - Storage: Memory, Redis, or DB
   - Dimensions: User, IP, endpoint, etc.

4. **User Experience Counts**
   - Clear error messages
   - Helpful headers
   - Retry guidance
   - Documentation

### Remember

> "Rate limiting is not about blocking users. It's about protecting your system so it can serve all users reliably. Choose the algorithm that fits your traffic patterns, implement it consistently, and communicate clearly with your users."

---

## Conclusion

Rate limiting is a critical component of any production system. By understanding these five algorithms and their trade-offs, you can choose the right approach for your needs:

- **Learning**: Start with Fixed Window Counter
- **Production**: Use Token Bucket or Sliding Window Counter  
- **High Traffic**: Sliding Window Counter with Redis
- **Strict Accuracy**: Sliding Window Log (if traffic allows)
- **Traffic Shaping**: Leaky Bucket

**Remember**: The best algorithm is the one that:
- Matches your traffic patterns
- Fits your resource constraints  
- Solves your specific problem
- Can be maintained by your team

---

**Version**: 6.0  
**Last Updated**: 2024  
**Part**: 6 of System Design Series  
**Topic**: Rate Limiting Algorithms  

---

*"Rate limiting is like a bouncer at a club. You want to let in as many people as the venue can safely hold, but no more. The algorithms are different strategies for counting and managing the crowd."*

Happy Learning! 🚀