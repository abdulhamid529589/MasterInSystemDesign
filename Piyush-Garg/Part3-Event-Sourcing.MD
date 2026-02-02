# System Design Part 3: Event Sourcing

## Table of Contents
- [Introduction](#introduction)
- [Understanding Events](#understanding-events)
- [Traditional CRUD vs Event Sourcing](#traditional-crud-vs-event-sourcing)
- [Problems with Traditional State Management](#problems-with-traditional-state-management)
- [Event Sourcing Explained](#event-sourcing-explained)
- [Core Concepts](#core-concepts)
  - [Event Logs](#event-logs)
  - [Hydration](#hydration)
  - [Reconciliation](#reconciliation)
  - [Time Machine Capability](#time-machine-capability)
- [Implementation Details](#implementation-details)
- [Apache Kafka and Event Sourcing](#apache-kafka-and-event-sourcing)
- [CQRS Pattern](#cqrs-pattern)
- [Real-World Examples](#real-world-examples)
- [Pros and Cons](#pros-and-cons)
- [Best Practices](#best-practices)

---

## Introduction

**Event Sourcing** is a software design pattern where changes to application state are stored as a sequence of events rather than just the current state. Instead of updating data directly, each state-changing action is recorded as an event in an append-only log.

**Key Principle**: Events become the source of truth, not the database state.

### Why Event Sourcing Matters

Event Sourcing is used by companies like:
- Uber (ride tracking)
- Netflix (user activity)
- Amazon (order processing)
- Banking systems (transaction history)

---

## Understanding Events

### What is an Event?

An event is any action performed by users on your platform.

**Amazon Example Events:**
```
1. Item Added to Cart
2. Checkout Initiated
3. Payment Processed
4. Order Placed
5. Product Uploaded (by seller)
6. Price Updated
7. Order Shipped
8. Order Delivered
```

**Key Characteristics:**
- Events represent **what happened**
- Events are **immutable** (cannot be changed)
- Events have **timestamps**
- Events are **sequential**

---

## Traditional CRUD vs Event Sourcing

### Traditional CRUD Approach

**Architecture:**
```
User → API Gateway → Server → Database
                                  ↓
                            Update State Directly
```

**Example Flow:**
```
1. User updates product price: $90 → $100
2. Server executes: UPDATE products SET price = 100 WHERE id = 1
3. Database state: price = 100
4. Previous price ($90) is LOST forever
```

**Problems:**
1. ❌ No history of changes
2. ❌ Can't track who made changes
3. ❌ Can't rollback to previous state
4. ❌ No audit trail
5. ❌ Race conditions on frequent updates

---

## Problems with Traditional State Management

### Problem 1: Database Bottleneck

**Scenario:**
```
Multiple Updates + Multiple Reads = Database Lock Contention

Writer 1: UPDATE product SET price = 100
Writer 2: UPDATE product SET price = 105  ← Must wait
Reader:   SELECT * FROM product           ← Must wait
```

**Issues:**
- Frequent updates acquire locks
- Readers blocked during writes
- Performance degradation
- Race conditions

### Problem 2: State Management Complexity

**Video Processing Pipeline Example:**

**Traditional Approach:**
```python
# Pseudo-code showing the problem

# Case 1: User uploads video
db.update(video_id, status="uploaded")

# Case 2: Worker picks up video
db.update(video_id, status="processing")

# Case 3: Processing complete
re_upload_processed_video()
db.update(video_id, status="success")
```

**What Can Go Wrong:**

```
Scenario 1: Database Busy
✅ Video uploaded successfully
❌ db.update() fails (database timeout)
Result: Video uploaded but status stuck at "null"

Scenario 2: Out of Sync
✅ Worker processing video
❌ db.update() fails
Result: Video processing but status stuck at "uploaded"

Scenario 3: Wrong Final State
✅ Processing complete
✅ Re-upload successful
❌ db.update() fails
Result: Video ready but status stuck at "processing"
```

**The Core Problem:**
- Reality ≠ Database State
- No way to track what actually happened
- No way to recover or debug
- Lost intermediate states

---

## Event Sourcing Explained

### The Paradigm Shift

**Instead of:**
```
Store: Current State (single value)
Example: status = "success"
```

**Event Sourcing:**
```
Store: Sequence of Events (append-only log)
Example:
  - VideoUploaded (10:00 AM)
  - ProcessingStarted (10:05 AM)
  - ProcessingComplete (10:10 AM)
```

### Core Principle

> **Don't update state directly. Instead, emit events. Derive state from events.**

---

## Core Concepts

### Event Logs

**Characteristics:**
1. **Append-only**: New events added to end
2. **Immutable**: Cannot modify existing events
3. **Ordered**: Chronological sequence matters
4. **Complete history**: Every change captured

**Event Structure:**
```json
{
  "eventId": "evt_001",
  "eventType": "VideoUploaded",
  "timestamp": "2024-02-01T10:00:00Z",
  "data": {
    "videoId": "vid_123",
    "path": "s3://bucket/raw/video.mp4",
    "uploadedBy": "user_456"
  }
}
```

**Storage:**
```
Event Stream → S3 / Database / Kafka
  - Always preserved
  - Never deleted
  - Source of truth
```

### Hydration

**Definition**: Process of reconstructing current state by replaying events

**How It Works:**

```
Events:
1. VideoUploaded (10:00)
2. ProcessingStarted (10:05)
3. ProcessingComplete (10:10)

Hydration Process:
Initial State: null
↓ Apply Event 1
State: "uploaded"
↓ Apply Event 2
State: "processing"
↓ Apply Event 3
Final State: "success"

Result: Return "success" to user
```

**Example: Banking Account Balance**

```
Events:
1. InitialDeposit: $500
2. Deposit: $200
3. Deposit: $300
4. Withdraw: $500

Hydration:
$500 (initial)
+ $200 → $700
+ $300 → $1000
- $500 → $500

Current Balance: $500
```

**Key Point**: Never store balance directly. Always compute from events.

### Reconciliation

**Purpose**: Verify state correctness when users complain

**Scenario:**
```
User: "My balance should be $300, not $500!"

Reconciliation Process:
1. Fetch all events for user
2. Replay events in sequence
3. Verify each calculation
4. Show audit trail to user:
   - Initial: $500
   - Deposit: $200 (Transaction #123)
   - Deposit: $300 (Transaction #456)
   - Withdraw: $500 (Transaction #789)
   Final: $500 ✓
```

**Benefits:**
- Complete audit trail
- Dispute resolution
- Debugging capability
- Compliance (financial regulations)

### Time Machine Capability

**Question**: "What was my balance 1 month ago?"

**Traditional Database:**
```
❌ Cannot answer - only current state stored
```

**Event Sourcing:**
```
✅ Replay events up to that date

Events (filtered by date ≤ 1 month ago):
1. InitialDeposit: $500 (Jan 1)
2. Deposit: $200 (Jan 15)

Balance on Jan 31: $700
```

**Use Cases:**
- Historical reporting
- Regulatory compliance
- Point-in-time recovery
- Trend analysis

---

## Implementation Details

### Architecture

```
┌─────────────────────────────────────────────┐
│                   User                      │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│              API Gateway                    │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│         Application Server                  │
│  (Emits Events, NOT Database Updates)       │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│         Event Stream (Kafka/SQS)            │
│         Append-Only Event Log               │
└─────────────────┬───────────────────────────┘
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
┌─────────────┐      ┌─────────────┐
│  Event      │      │  Event      │
│  Processor  │      │  Storage    │
│  (Workers)  │      │  (S3/DB)    │
└──────┬──────┘      └─────────────┘
       │
       ↓
┌─────────────┐
│  Read Model │
│  (Cache)    │
│  PostgreSQL │
└─────────────┘
```

### Video Processing Pipeline Example

**Event-Based Implementation:**

```javascript
// Instead of updating database directly
// Emit events

// Event 1: Video Uploaded
emitEvent({
  type: "VideoUploaded",
  data: {
    videoId: "vid_123",
    path: "s3://raw/video.mp4"
  }
});

// Event 2: Processing Started
emitEvent({
  type: "ProcessingStarted",
  data: {
    videoId: "vid_123",
    workerId: "worker_5"
  }
});

// Event 3: Processing 50% Complete
emitEvent({
  type: "ProcessingProgress",
  data: {
    videoId: "vid_123",
    progress: 50
  }
});

// Event 4: Processing Success
emitEvent({
  type: "ProcessingComplete",
  data: {
    videoId: "vid_123",
    outputPath: "s3://processed/video.mp4"
  }
});
```

**Event Processor:**
```javascript
// Worker processes events
eventStream.on('event', async (event) => {
  switch(event.type) {
    case 'VideoUploaded':
      // Add to processing queue
      break;
    case 'ProcessingStarted':
      // Update cache state to "processing"
      await cache.set(`video:${event.data.videoId}`, {
        status: 'processing',
        worker: event.data.workerId
      });
      break;
    case 'ProcessingComplete':
      // Update cache state to "success"
      await cache.set(`video:${event.data.videoId}`, {
        status: 'success',
        output: event.data.outputPath
      });
      break;
  }
});
```

### Optimizing Reads with Caching

**Problem**: Replaying thousands of events for every read is inefficient

**Solution**: Maintain a read-optimized cache

```
Event Stream (Source of Truth)
       ↓
   Processor
       ↓
  Cache/Read Model (PostgreSQL)
       ↓
   Fast Reads
```

**How It Works:**

```javascript
// Background process continuously updates cache
async function updateReadModel(event) {
  const currentState = await getFromCache(event.videoId);
  const newState = applyEvent(currentState, event);
  await saveToCache(event.videoId, newState);
}

// User queries are fast
async function getVideoStatus(videoId) {
  // Read from cache, not event replay
  return await cache.get(`video:${videoId}`);
}

// If user complains, reconcile
async function reconcileVideo(videoId) {
  const events = await getEventsForVideo(videoId);
  const recomputedState = replayEvents(events);
  
  // Compare with cache
  const cachedState = await cache.get(`video:${videoId}`);
  
  if (recomputedState !== cachedState) {
    // Fix cache
    await cache.set(`video:${videoId}`, recomputedState);
    console.log('State reconciled:', recomputedState);
  }
  
  return recomputedState;
}
```

---

## Apache Kafka and Event Sourcing

### The Out-of-Order Problem

**Scenario: 3 Workers Processing Events**

```
Event Stream:
1. VideoUploaded
2. ProcessingStarted
3. ProcessingComplete

Worker Distribution:
Worker 1: Picks event #1 (slow, under load)
Worker 2: Picks event #2 → Updates state to "processing"
Worker 3: Picks event #3 → Updates state to "success"

Then Worker 1 finishes:
Worker 1: Updates state to "uploaded" ❌

Final State: "uploaded" (WRONG!)
Should be: "success"
```

**Result**: Events processed out of order = corrupt state

### Kafka's Solution: Consumer Groups & Partitions

**Kafka Architecture:**

```
Topic: video-events
Partitions:
  - Partition 0
  - Partition 1
  - Partition 2

Consumer Group: video-processors
Consumers:
  - Worker 1
  - Worker 2
  - Worker 3
```

**Partition Assignment:**

```
Video A events → Partition 0 → Worker 1 (always)
Video B events → Partition 1 → Worker 2 (always)
Video C events → Partition 2 → Worker 3 (always)
```

**Guarantee**: All events for same video go to same worker = order preserved

**How Partitioning Works:**

```python
# Partition key = videoId
partition = hash(videoId) % num_partitions

# All events for vid_123 go to same partition
emitEvent({
  partitionKey: "vid_123",  # Important!
  type: "VideoUploaded",
  data: {...}
})
```

**Result:**
```
Video A events (in partition 0):
1. VideoUploaded → Worker 1
2. ProcessingStarted → Worker 1
3. ProcessingComplete → Worker 1

Sequential processing guaranteed ✓
Even if Worker 1 is slow, order maintained
```

### Kafka Consumer Groups

```
Consumer Group: video-processor-group
├─ Worker 1 → Subscribes to Partition 0, 3, 6
├─ Worker 2 → Subscribes to Partition 1, 4, 7
└─ Worker 3 → Subscribes to Partition 2, 5, 8

Guarantees:
✓ Each partition consumed by exactly one worker
✓ Events in partition processed in order
✓ Parallel processing across partitions
```

---

## CQRS Pattern

### Command Query Responsibility Segregation

**Definition**: Separate read and write operations into different models

**Traditional Approach:**
```
Same Database:
  - Handle Writes (updates)
  - Handle Reads (queries)
  - Same model for both

Problem: Conflicting requirements
```

**CQRS Approach:**
```
Write Model (Commands):
  - Optimized for writes
  - Event sourcing
  - Append-only log

Read Model (Queries):
  - Optimized for reads
  - Denormalized
  - Fast lookups
  - Built from events
```

### CQRS + Event Sourcing Architecture

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     ├─── Write (Command) ───────────────────┐
     │                                       ↓
     │                           ┌───────────────────┐
     │                           │  Command Handler  │
     │                           └─────────┬─────────┘
     │                                     ↓
     │                           ┌───────────────────┐
     │                           │   Event Store     │
     │                           │  (Source of Truth)│
     │                           └─────────┬─────────┘
     │                                     ↓
     │                           ┌───────────────────┐
     │                           │ Event Processors  │
     │                           └─────────┬─────────┘
     │                                     ↓
     └─── Read (Query) ─────────→  ┌─────────────────┐
                                   │   Read Model    │
                                   │ (Optimized DB)  │
                                   └─────────────────┘
```

**Benefits:**
1. **Independent Scaling**: Scale reads and writes separately
2. **Optimized Performance**: Each model optimized for its purpose
3. **Flexibility**: Different databases for reads vs writes
4. **Better Architecture**: Clear separation of concerns

**Example:**

```javascript
// Write Side (Commands)
async function updatePrice(productId, newPrice) {
  // Emit event
  await eventStore.append({
    type: 'PriceUpdated',
    productId,
    newPrice,
    timestamp: Date.now()
  });
}

// Read Side (Queries)
async function getProduct(productId) {
  // Read from optimized read model
  return await readModel.query(
    'SELECT * FROM products WHERE id = ?',
    [productId]
  );
}

// Background: Event processor updates read model
eventStore.on('PriceUpdated', async (event) => {
  await readModel.update(
    'UPDATE products SET price = ? WHERE id = ?',
    [event.newPrice, event.productId]
  );
});
```

---

## Real-World Examples

### Amazon Order Processing

**Traditional Way (WRONG):**
```sql
UPDATE orders SET status = 'delivered' WHERE id = 123
```

**Event Sourcing Way (CORRECT):**
```
Events:
1. OrderPlaced (11:00 AM)
2. PaymentProcessed (11:01 AM)
3. OrderPacked (2:00 PM)
4. OrderShipped (3:00 PM)
5. OutForDelivery (9:00 AM next day)
6. OrderDelivered (2:00 PM)
```

**Benefits:**
- Complete audit trail
- Track exact time of each step
- Dispute resolution
- Analytics (avg delivery time)
- Debugging (where orders get stuck)

### Banking Transaction System

**Events:**
```json
[
  {
    "type": "AccountOpened",
    "timestamp": "2024-01-01T09:00:00Z",
    "data": { "accountId": "acc_123", "initialBalance": 500 }
  },
  {
    "type": "MoneyDeposited",
    "timestamp": "2024-01-15T10:30:00Z",
    "data": { "accountId": "acc_123", "amount": 200, "transactionId": "txn_456" }
  },
  {
    "type": "MoneyWithdrawn",
    "timestamp": "2024-01-20T14:00:00Z",
    "data": { "accountId": "acc_123", "amount": 150, "transactionId": "txn_789" }
  }
]
```

**Query: Current Balance**
```javascript
function getCurrentBalance(accountId) {
  const events = getEventsForAccount(accountId);
  
  let balance = 0;
  for (const event of events) {
    switch(event.type) {
      case 'AccountOpened':
        balance = event.data.initialBalance;
        break;
      case 'MoneyDeposited':
        balance += event.data.amount;
        break;
      case 'MoneyWithdrawn':
        balance -= event.data.amount;
        break;
    }
  }
  
  return balance; // 550
}
```

**Query: Balance on Specific Date**
```javascript
function getBalanceOnDate(accountId, targetDate) {
  const events = getEventsForAccount(accountId)
    .filter(e => e.timestamp <= targetDate);
  
  return calculateBalance(events);
}

// Example: Balance on Jan 16, 2024
getBalanceOnDate('acc_123', '2024-01-16') // 700
```

### Uber Ride Tracking

**Events:**
```
1. RideRequested
2. DriverAssigned
3. DriverEnRoute
4. DriverArrived
5. RideStarted
6. RideInProgress (with location updates)
7. RideCompleted
8. PaymentProcessed
9. RideRated
```

**Benefits:**
- Real-time tracking
- Complete ride history
- Dispute resolution
- Driver performance analytics
- Surge pricing calculations

---

## Pros and Cons

### Advantages

**1. Complete Audit Trail**
```
Every action recorded
When, who, what changed
Regulatory compliance (SOX, GDPR)
```

**2. Time Travel**
```
State at any point in history
"What was inventory on Dec 31?"
Historical reports
Trend analysis
```

**3. Debugging & Recovery**
```
Reproduce bugs
Identify where things went wrong
Replay events to fix state
No data loss
```

**4. Event Replay**
```
Add new features retroactively
Example: Add "total lifetime spent" metric
Replay all purchase events
Calculate from beginning
```

**5. Scalability**
```
Append-only writes (fast)
No update locks
Parallel event processing
CQRS: Independent read/write scaling
```

**6. Business Intelligence**
```
Complete user journey
A/B testing analysis
Feature usage tracking
Customer behavior insights
```

### Disadvantages

**1. Complexity**
```
❌ Steep learning curve
❌ More code to maintain
❌ Requires architectural shift
❌ Team training needed
```

**2. Storage Costs**
```
❌ Every event stored forever
❌ Storage grows continuously
❌ Example: 1M events/day = 365M/year
❌ Need archival strategy
```

**3. Event Schema Evolution**
```
❌ Cannot change past events
❌ Must support old event versions
❌ Migration complexity
❌ Versioning required

Example:
v1: { "price": 100 }
v2: { "price": { "amount": 100, "currency": "USD" } }

Must handle both during replay!
```

**4. Eventual Consistency**
```
❌ Event processing takes time
❌ Read model may be stale
❌ Users see delayed updates
❌ Need to communicate delay

Example:
User updates email → Event emitted (instant)
→ Event processed (5 seconds later)
→ Read model updated (10 seconds later)
User refreshes → Sees old email (confusing!)
```

**5. Learning Curve**
```
❌ Different from CRUD
❌ Requires mindset shift
❌ Tools and libraries immature
❌ Debugging is different
```

**6. Cannot Delete Data**
```
❌ GDPR "right to be forgotten"
❌ Events are immutable
❌ Workaround: Encryption + delete keys
❌ Compliance challenges
```

**7. Query Complexity**
```
❌ Complex queries need materialized views
❌ Cannot do ad-hoc SQL easily
❌ Need to maintain multiple read models
❌ More infrastructure
```

---

## Best Practices

### 1. Event Naming Conventions

**Use Past Tense:**
```
✅ OrderPlaced
✅ PaymentProcessed
✅ UserRegistered

❌ PlaceOrder (command, not event)
❌ ProcessPayment (command, not event)
```

**Be Specific:**
```
✅ EmailAddressChanged
❌ UserUpdated (too vague)

✅ ProductPriceIncreased
❌ ProductChanged (what changed?)
```

### 2. Event Structure

```javascript
{
  // Required fields
  "eventId": "uuid",
  "eventType": "OrderPlaced",
  "timestamp": "ISO-8601",
  "aggregateId": "order_123",  // What object
  "version": 1,                // Event schema version
  
  // Metadata
  "metadata": {
    "userId": "user_456",
    "correlationId": "correlation_789",
    "causationId": "event_that_caused_this"
  },
  
  // Event-specific data
  "data": {
    "orderId": "order_123",
    "items": [...],
    "total": 99.99
  }
}
```

### 3. Partition Strategy

```javascript
// Use entity ID as partition key
function getPartitionKey(event) {
  // All events for same order go to same partition
  return event.data.orderId;
  
  // Or for user events
  return event.data.userId;
}

// Ensures:
// - Ordered processing per entity
// - Parallel processing across entities
```

### 4. Snapshot Strategy

**Problem**: Replaying 1 million events for each query is slow

**Solution**: Periodic snapshots

```javascript
// Create snapshot every 100 events
if (eventCount % 100 === 0) {
  await saveSnapshot(aggregateId, currentState);
}

// To rebuild state:
async function getCurrentState(aggregateId) {
  // 1. Get latest snapshot
  const snapshot = await getLatestSnapshot(aggregateId);
  
  // 2. Get events after snapshot
  const events = await getEventsSince(
    aggregateId,
    snapshot.timestamp
  );
  
  // 3. Replay only recent events
  return replayEvents(snapshot.state, events);
}

// Example:
// Total events: 1000
// Snapshot at event 900 with state
// Replay only events 901-1000 (100 events)
// Much faster than replaying all 1000!
```

### 5. Event Versioning

```javascript
// Event processors handle multiple versions
async function processEvent(event) {
  switch(event.version) {
    case 1:
      return processV1Event(event);
    case 2:
      return processV2Event(event);
    default:
      throw new Error(`Unknown version: ${event.version}`);
  }
}

// Upcasting: Convert old events to new format
function upcastEvent(oldEvent) {
  if (oldEvent.version === 1) {
    return {
      ...oldEvent,
      version: 2,
      data: {
        ...oldEvent.data,
        currency: oldEvent.data.currency || 'USD'
      }
    };
  }
  return oldEvent;
}
```

### 6. Idempotency

**Ensure events can be processed multiple times safely:**

```javascript
async function processEvent(event) {
  // Check if already processed
  const processed = await isEventProcessed(event.eventId);
  if (processed) {
    console.log('Event already processed, skipping');
    return;
  }
  
  // Process event
  await updateReadModel(event);
  
  // Mark as processed
  await markEventProcessed(event.eventId);
}
```

### 7. Error Handling

```javascript
// Dead Letter Queue for failed events
async function processEvent(event) {
  try {
    await handleEvent(event);
  } catch (error) {
    event.retryCount = (event.retryCount || 0) + 1;
    
    if (event.retryCount < 3) {
      // Retry
      await requeueEvent(event);
    } else {
      // Move to Dead Letter Queue
      await moveToDeadLetterQueue(event, error);
      await alertOps('Event processing failed', event);
    }
  }
}
```

### 8. Monitoring

```javascript
// Track important metrics
metrics.increment('events.published', {
  type: event.eventType
});

metrics.timing('event.processing.duration', duration);

metrics.gauge('event.queue.size', queueSize);

metrics.increment('event.processing.failed', {
  type: event.eventType,
  error: error.type
});
```

---

## When to Use Event Sourcing

### Good Use Cases

**✅ Financial Systems**
- Banking transactions
- Payment processing
- Accounting systems
- Audit requirements

**✅ Order Management**
- E-commerce orders
- Fulfillment tracking
- Inventory management
- Supply chain

**✅ Collaborative Systems**
- Document editing (Google Docs)
- Project management
- Version control
- Wikis

**✅ Complex Workflows**
- Approval processes
- State machines
- Multi-step procedures
- Regulatory compliance

**✅ Analytics-Heavy**
- User behavior tracking
- A/B testing platforms
- Recommendation engines
- Business intelligence

### Bad Use Cases

**❌ Simple CRUD Apps**
- Basic blog
- Simple todo list
- Contact management
- Over-engineering

**❌ Read-Heavy, Simple Writes**
- News website
- Documentation site
- Static content
- Unnecessary complexity

**❌ Short-Lived Data**
- Session data
- Temporary caches
- Chat messages (maybe)
- High volume, low value

**❌ Small Teams Without Experience**
- Startup MVP
- Proof of concept
- Small team
- Time constraints

---

## Migration Strategy

### From CRUD to Event Sourcing

**Phase 1: Hybrid Approach**
```
Keep existing database
Add event log alongside
Dual writes:
  - Write to database (as before)
  - Emit event (new)
  
Benefits:
- Low risk
- Gradual migration
- Rollback possible
```

**Phase 2: Event-First**
```
Emit events first
Update database from events
Read from database still

Benefits:
- Events become source of truth
- Can rebuild state from events
- Database becomes cache
```

**Phase 3: Full Event Sourcing**
```
Read from materialized views
Database built entirely from events
Can delete and rebuild database

Benefits:
- True event sourcing
- Maximum benefits
- Full audit trail
```

---

## Tools and Technologies

### Event Stores

**Specialized Event Stores:**
- **EventStoreDB**: Purpose-built for event sourcing
- **Apache Kafka**: Distributed event streaming
- **AWS EventBridge**: Serverless event bus
- **Azure Event Hubs**: Cloud event ingestion

**Database Options:**
- **PostgreSQL**: With JSONB for events
- **MongoDB**: Document store for events
- **Cassandra**: High-throughput writes
- **DynamoDB**: Serverless option

### Frameworks

**Java:**
- Axon Framework
- Eventuate

**JavaScript/TypeScript:**
- NestJS + CQRS
- Eventuous
- Node Event Store

**.NET:**
- MediatR
- EventFlow
- NEventStore

**Python:**
- Eventsourcing library
- Axon Server

---

## Comparison Table

| Aspect | Traditional CRUD | Event Sourcing |
|--------|-----------------|----------------|
| **Storage** | Current state only | All events |
| **History** | Lost on update | Complete |
| **Audit Trail** | Manual logging | Built-in |
| **Time Travel** | Impossible | Easy |
| **Complexity** | Simple | Complex |
| **Debugging** | Difficult | Easier |
| **Performance (Write)** | Fast | Very Fast (append-only) |
| **Performance (Read)** | Fast | Need cache |
| **Storage Cost** | Low | Higher |
| **Compliance** | Manual | Automatic |
| **Scalability** | Vertical | Horizontal |

---

## Real-World Architecture Example

### E-Commerce Order System

```
┌──────────────────────────────────────────────────────┐
│                    Client App                        │
└────────────────────┬─────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────┐
│                  API Gateway                         │
│              (Rate Limiting, Auth)                   │
└────────────────────┬─────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ↓                         ↓
┌─────────────────┐      ┌─────────────────┐
│ Command Service │      │  Query Service  │
│  (Write Model)  │      │  (Read Model)   │
└────────┬────────┘      └────────┬────────┘
         │                        │
         ↓                        ↓
┌─────────────────┐      ┌─────────────────┐
│  Event Store    │      │  Read Database  │
│   (Kafka)       │      │  (PostgreSQL)   │
└────────┬────────┘      └─────────────────┘
         │
         ↓
┌─────────────────────────────────────────┐
│         Event Processors                │
├─────────────────────────────────────────┤
│ • Update Read Model                     │
│ • Send Notifications                    │
│ • Update Inventory                      │
│ • Generate Analytics                    │
│ • Trigger Workflows                     │
└─────────────────────────────────────────┘
```

**Event Flow:**

```
1. User places order
   → OrderPlaced event emitted

2. Event processors triggered:
   - Update read model (order visible to user)
   - Send email notification
   - Reserve inventory
   - Charge payment
   - Update analytics

3. Payment succeeds
   → PaymentProcessed event emitted

4. Inventory reserved
   → InventoryReserved event emitted

5. Order shipped
   → OrderShipped event emitted

6. User queries order status
   → Read from optimized read model (fast)

7. User disputes charge
   → Replay all events
   → Show complete audit trail
   → Resolve dispute
```

---

## Key Takeaways

### Core Principles

1. **Events are Truth**
   - Never delete events
   - Events are immutable
   - State derived from events

2. **Append-Only**
   - Only add new events
   - Never modify existing events
   - Sequence matters

3. **Replay Capability**
   - Rebuild state anytime
   - Time travel possible
   - Complete audit trail

4. **Eventual Consistency**
   - Embrace async
   - Read models lag behind
   - Acceptable for most use cases

### When to Consider Event Sourcing

**Yes, if you need:**
- ✅ Complete audit trail
- ✅ Regulatory compliance
- ✅ Historical analysis
- ✅ Complex workflows
- ✅ Event-driven architecture

**No, if you have:**
- ❌ Simple CRUD requirements
- ❌ Small team with no experience
- ❌ Time/budget constraints
- ❌ No audit requirements
- ❌ Mostly read operations

### Remember

> "Event Sourcing is not a silver bullet. It solves specific problems related to audit trails, historical queries, and complex business logic. But it introduces complexity. Choose wisely based on your actual requirements, not because it's trendy."

---

## Additional Resources

### Reading

- **Martin Fowler**: [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- **Microsoft**: [Event Sourcing Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- **AWS**: [Event Sourcing with AWS](https://aws.amazon.com/blogs/compute/event-sourcing-with-aws-lambda/)

### Videos

- Greg Young - CQRS and Event Sourcing
- Martin Fowler - Event Sourcing
- Kafka Summit talks

### Books

- **"Domain-Driven Design"** by Eric Evans
- **"Implementing Domain-Driven Design"** by Vaughn Vernon
- **"Microservices Patterns"** by Chris Richardson

### Practice

- Build a simple banking app with event sourcing
- Implement order processing with events
- Create audit trail for user actions

---

## Conclusion

Event Sourcing is a powerful pattern that:
- Provides complete audit trails
- Enables time travel queries
- Simplifies debugging
- Supports complex business logic
- Enables event-driven architectures

But it requires:
- Architectural commitment
- Team expertise
- Careful implementation
- Proper tooling
- Clear requirements

**Next Steps:**
1. Understand when it's needed
2. Start with hybrid approach
3. Use proven tools (Kafka)
4. Implement CQRS alongside
5. Monitor and optimize

**Coming Next**: CQRS (Command Query Responsibility Segregation) in depth

---

**Version**: 3.0  
**Last Updated**: 2024  
**Part**: 3 of System Design Series  
**Topic**: Event Sourcing  
**Related**: CQRS, Event-Driven Architecture, Kafka

---

*"Every action in Amazon, Uber, Netflix is an event. They don't update database states directly. They emit events, replay them, and derive current state. That's the power of Event Sourcing."*

Happy Learning! 🚀