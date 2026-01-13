# System Design: Essential Terminology & Concepts

## Overview

This guide provides a comprehensive introduction to the most important terminology and concepts in system design. This is foundational knowledge that will prepare you for understanding how to scale applications from a few users to millions of users.

**Learning Approach**: This guide gives you a thousand-foot view of essential topics. You don't need to master every concept in depth immediately—the goal is to familiarize yourself with these terms so they're not completely new when you encounter them in practice.

---

## Table of Contents

1. [Distributed Systems](#distributed-systems)
2. [Availability](#availability)
3. [Reliability](#reliability)
4. [Consistency](#consistency)
5. [Scalability vs Elasticity](#scalability-vs-elasticity)
6. [Fault Tolerance & Redundancy](#fault-tolerance--redundancy)
7. [Database Concepts](#database-concepts)
8. [ACID Properties](#acid-properties)
9. [Database Partitioning & Sharding](#database-partitioning--sharding)
10. [Caching Introduction](#caching-introduction)
11. [Key Takeaways](#key-takeaways)

---

## Distributed Systems

### Definition

**Distributed Computing**: Running an application across multiple computers instead of a single machine, where all computers work together to provide a single result.

### Visualization

```
┌─────────────────────────────────────┐
│     Distributed System              │
│  ┌──────────┐  ┌──────────┐        │
│  │Computer 1│  │Computer 2│        │
│  └──────────┘  └──────────┘        │
│       ┌──────────┐                  │
│       │Computer 3│                  │
│       └──────────┘                  │
│                                     │
│  All working together → Single Result │
└─────────────────────────────────────┘
```

### Key Characteristics

- Applications run on multiple machines simultaneously
- All computers must coordinate to produce unified results
- Applies to any service: backends, databases, caching layers, etc.
- Introduces complexity but enables scalability and fault tolerance

### Challenges Introduced

- Coordination between machines
- Data synchronization
- Network communication overhead
- Handling node failures
- Maintaining consistency across nodes

---

## Availability

### Definition

**Availability**: A system's ability to remain accessible and operational. In distributed systems, if one node fails, other nodes compensate by taking its load.

### High Availability System

```
     User Request
          ↓
    ┌─────────────┐
    │   Service   │
    └─────────────┘
         ↓
    ┌────┴────┬────────┐
    ↓         ↓        ↓
┌───────┐ ┌───────┐ ┌───────┐
│Node 1 │ │Node 2 │ │Node 3 │
└───────┘ └───────┘ └───────┘
    ✓         ✓        ✗ (DOWN)
    
Request automatically routes to Node 1 or Node 2
```

### Measuring Availability: The Nines

Availability is measured in "nines" representing uptime percentage:

| Nines | Uptime | Downtime Per Year | Assessment |
|-------|--------|-------------------|------------|
| 1 nine (90%) | 90% | 36.5 days | Unacceptable |
| 2 nines (99%) | 99% | 3.65 days | Poor |
| 3 nines (99.9%) | 99.9% | 8.77 hours | Acceptable for some apps |
| 4 nines (99.99%) | 99.99% | 52.6 minutes | **Recommended minimum** |
| 5 nines (99.999%) | 99.999% | 5.26 minutes | High availability |
| 6 nines (99.9999%) | 99.9999% | 31.5 seconds | Mission critical |

### Recommended Baseline

**Four Nines (99.99%)** = Less than 1 hour of downtime per year

This is a good starting point for most applications. More nines may be necessary for:
- Medical systems
- Financial systems
- Military systems
- Mission-critical infrastructure

### Trade-offs

```
More Nines (↑)
    ↓
Higher Availability ✓
    BUT
Exponentially Higher Costs ✗
Increased Implementation Complexity ✗
More Skilled Developers Required ✗
```

---

## Reliability

### Definition

**Reliability**: A service's ability to perform its intended function correctly. A system can be highly available but unreliable if it's accessible but not functioning properly.

### Availability vs Reliability

Think of it like a friend:

**Available but Unreliable Friend**:
- Always answers phone calls ✓
- Always texts back ✓
- Shows up when you need help ✓
- But... doesn't actually help or makes things worse ✗

**System Example**:
```
Website: Available ✓ (responds to requests)
Website: Unreliable ✗ (loses data, returns errors, behaves incorrectly)
```

### Key Questions for Reliability

- Does the application work as intended?
- Is data being stored correctly?
- Are calculations accurate?
- Do features function properly?
- Is data integrity maintained?

### Measuring Reliability

Unlike availability (which uses "nines"), reliability is more abstract:
- Define what "success" means for your application
- Define what "failure" looks like
- Monitor error rates, data integrity, and functional correctness
- Can have 99.999% uptime but 0% reliability

---

## Consistency

### Definition

**Consistency**: All users see the same data at the same time, regardless of which server they access.

### Single Server = Always Consistent

```
    Client 1 ──┐
               ├─→ Single Server (Always same data)
    Client 2 ──┘
    
Result: 100% Consistent ✓
```

### Distributed System = Potential Inconsistency

```
Client 1 ──→ Server 1 (User: "Caleb") ✓
Client 2 ──→ Server 2 (User: "Caleb") ✓
Client 3 ──→ Server 3 (User: "John") ✗ OLD DATA

Scenario: User changed from "John" to "Caleb"
Problem: Server 3 hasn't received the update yet
Result: Inconsistent ✗
```

---

## Types of Consistency

### 1. Strong Consistency

**Definition**: When data changes, the change is immediately available on ALL nodes before being confirmed.

#### How It Works

```
Write to Node 1
    ↓
Pending (waiting...)
    ↓
Propagate to Node 2 → Applied
    ↓
Propagate to Node 3 → Applied
    ↓
ALL nodes updated
    ↓
Write CONFIRMED ✓

Timeline: [Change] ═══════════════════════> [Confirmed]
          (Wait for all nodes to sync)
```

#### Characteristics

✅ **Advantages**:
- All nodes always have identical data
- No stale reads
- Data integrity guaranteed

❌ **Disadvantages**:
- Increased latency (slower writes)
- Reduced write throughput
- Potential availability impact

#### Use Cases

**MUST USE for**:
- 💰 **Banking/Financial systems** (account balances, transactions)
- 🏥 **Medical systems** (real-time health records)
- ⚔️ **Military systems** (critical operational data)
- 🔒 **Any system where data inconsistency could cause harm**

---

### 2. Eventual Consistency

**Definition**: When data changes, the change is immediately confirmed on one node, then eventually propagates to others.

#### How It Works

```
Write to Node 1
    ↓
Immediately CONFIRMED ✓
    ↓
Propagate to Node 2 (in background)
    ↓
Propagate to Node 3 (in background)

Timeline: [Change] → [Confirmed]
          (Fast)    ↓
                    Sync happens later...
```

#### Characteristics

✅ **Advantages**:
- Fast write operations
- Higher throughput
- Lower latency
- Better write performance

❌ **Disadvantages**:
- Temporary inconsistency possible
- Clients may see stale data
- Not suitable for critical operations

#### The Inconsistency Window

```
Time: 0ms - User updates name to "Caleb"
Time: 10ms - Node 1 has "Caleb" ✓
Time: 50ms - Node 2 has "Caleb" ✓
Time: 100ms - Node 3 still has "John" ✗

During this window (0-100ms):
- Some users see "Caleb"
- Other users see "John"
- Eventually all see "Caleb"
```

#### Use Cases

**APPROPRIATE for**:
- 📱 **Social media** (posts, likes, comments)
- 📝 **Basic CRUD apps** (Create, Read, Update, Delete)
- 📰 **News feeds**
- 💬 **Chat applications** (non-critical messages)
- 🛒 **E-commerce product catalogs**
- **Any system where brief delays are acceptable**

---

## Consistency Comparison Summary

| Aspect | Strong Consistency | Eventual Consistency |
|--------|-------------------|---------------------|
| **Write Speed** | Slower ⏱️ | Faster ⚡ |
| **Data Accuracy** | Always accurate ✓ | Temporarily stale ⚠️ |
| **Complexity** | Higher | Lower |
| **Use Case** | Financial, Medical | Social, General apps |
| **Availability Impact** | Can reduce availability | Better availability |

---

## Scalability vs Elasticity

### Scalability

**Definition**: An application's ability to support increased workload without breaking down.

#### Measuring Scalability

```
Server Capacity Visualization:

Max Capacity: ████████████████████ 100%
                                    ↑ Bad (approaching limit)
Peak Usage:   ████████████░░░░░░░░ 60%
                                    ↑ Good (headroom available)
Typical Usage: ████░░░░░░░░░░░░░░░ 20%
                                    ↑ Excellent
```

#### Key Principle

**Headroom**: Always maintain space between current usage and maximum capacity.

**Consider**:
- Day-to-day typical usage
- Peak usage times (8am rush, lunch, evening)
- Seasonal spikes (Black Friday, holidays)
- Unexpected viral traffic

---

### Vertical Scaling (Scale Up)

**Definition**: Adding more resources to a single server.

```
Before:                  After:
┌──────────┐            ┌──────────┐
│  2 CPU   │    →       │  8 CPU   │
│  4GB RAM │            │ 32GB RAM │
│ 500GB HD │            │  2TB SSD │
└──────────┘            └──────────┘
  1 Server                1 Server
```

#### Characteristics

✅ **Advantages**:
- Simple to implement
- No architectural changes needed
- No networking complexity
- Easy to manage (single server)

❌ **Disadvantages**:
- Hardware limits (can't scale infinitely)
- Expensive at high end
- Still single point of failure
- Downtime required for upgrades

#### Typical Use Case
- Small to medium applications
- Quick temporary solution
- Database servers (often start here)

---

### Horizontal Scaling (Scale Out)

**Definition**: Adding more servers to distribute the workload.

```
Before:                  After:
┌──────────┐            ┌──────────┐
│  Server  │            │ Server 1 │
│          │    →       ├──────────┤
│          │            │ Server 2 │
└──────────┘            ├──────────┤
  1 Server              │ Server 3 │
                        └──────────┘
                         3 Servers
```

#### Characteristics

✅ **Advantages**:
- Nearly unlimited scaling potential
- Better fault tolerance
- More cost-effective at scale
- No single point of failure

❌ **Disadvantages**:
- Requires architectural changes
- More complex to manage
- Need load balancing
- Data synchronization challenges

#### Typical Use Case
- Large-scale applications
- High-traffic websites
- Distributed systems
- Modern cloud applications

---

### Combined Approach

```
Scale Vertically AND Horizontally:

┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ 8 CPU       │  │ 8 CPU       │  │ 8 CPU       │
│ 32GB RAM    │  │ 32GB RAM    │  │ 32GB RAM    │
│ Large Server│  │ Large Server│  │ Large Server│
└─────────────┘  └─────────────┘  └─────────────┘

Best of both worlds:
- Powerful individual servers
- Multiple servers for distribution
```

---

### Elasticity

**Definition**: Automatic adjustment of resources based on current workload—both scaling UP and DOWN.

#### Scalability vs Elasticity

**Scalability**:
```
Workload ↑ → Manually add resources → Handle load
```

**Elasticity**:
```
Workload ↑ → Automatically add resources → Handle load
            ↓
Workload ↓ → Automatically remove resources → Save cost
```

#### Elasticity in Action

```
Typical Day:

Morning (Low traffic):     [Server 1] [Server 2]
                          ↓
Afternoon (High traffic):  [Server 1] [Server 2] [Server 3] [Server 4]
                          ↓
Night (Low traffic):       [Server 1] [Server 2]

Resources automatically scale with demand
```

#### How Cloud Providers Enable Elasticity

Cloud providers like AWS, Google Cloud, Azure:
- Use **virtualization** to allocate/deallocate resources
- Have massive pools of physical servers
- Dynamically assign resources across applications
- No physical hardware changes needed
- Scale happens in seconds/minutes

#### Benefits

✅ **Cost Savings**: Pay only for what you use
✅ **Automatic**: No manual intervention needed  
✅ **Fast**: Resources adjust in real-time
✅ **Optimal**: Always right-sized for current load

---

### Summary Table

| Feature | Scalability | Elasticity |
|---------|------------|-----------|
| **Focus** | Handling increased load | Automatic resource adjustment |
| **Direction** | Usually just UP | Both UP and DOWN |
| **Timing** | Often manual/planned | Automatic/real-time |
| **Goal** | Prevent system failure | Optimize cost and performance |
| **Implementation** | Add resources | Cloud auto-scaling |

---

## Fault Tolerance & Redundancy

### Single Point of Failure (SPOF)

**Definition**: Any component in a system that, if it fails, causes the entire system to fail.

#### Example System with SPOF

```
┌─────────────┐
│  Frontend   │ ← Multiple servers ✓
│  (3 servers)│
└──────┬──────┘
       │
┌──────▼──────┐
│   Backend   │ ← Multiple servers ✓
│  (3 servers)│
└──────┬──────┘
       │
┌──────▼──────┐
│  Database   │ ← SINGLE server ✗ SPOF!
│  (1 server) │
└─────────────┘

Problem: If database goes down, entire system fails
```

### Redundancy

**Definition**: Having backup components ready to take over if primary components fail.

#### Key Concept

In computing, **redundancy is GOOD** ✓  
(Unlike in everyday language where redundant = unnecessary)

**Redundancy**:
- Increases reliability
- Eliminates single points of failure
- Enables high availability
- Worth the extra cost

---

### Database Redundancy Example

#### Without Redundancy (SPOF)

```
Backend → [Primary Database] ✗ If this fails, system down!
```

#### With Redundancy (Fault Tolerant)

```
Backend → [Primary Database] ←→ [Standby Database]
                                (Continuous replication)

If Primary fails:
Backend → [Primary Database 💥] 
                ↓
Backend → [Standby Database ✓] (Promoted to primary)
```

---

### Fault Tolerance

**Definition**: A system's ability to continue operating properly when one or more components fail.

#### Achieving Fault Tolerance

1. **Identify** single points of failure
2. **Add redundancy** to critical components
3. **Implement failover** mechanisms
4. **Test failure scenarios** regularly

#### Example: Fault-Tolerant Architecture

```
                    ┌───────────────┐
        ┌──────────→│  Frontend 1   │
        │           └───────────────┘
        │           ┌───────────────┐
User ───┼──────────→│  Frontend 2   │
        │           └───────────────┘
        │           ┌───────────────┐
        └──────────→│  Frontend 3   │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
        ┌──────────→│   Backend 1   │
        │           └───────────────┘
        │           ┌───────────────┐
        ├──────────→│   Backend 2   │
        │           └───────────────┘
        │           ┌───────────────┐
        └──────────→│   Backend 3   │
                    └───────┬───────┘
                            │
        ┌───────────────────┴───────────────────┐
        │                                        │
    ┌───▼──────────┐                   ┌────────▼─────┐
    │   Primary    │◄─── Replication ─→│   Standby    │
    │   Database   │                   │   Database   │
    └──────────────┘                   └──────────────┘

No single point of failure! ✓
```

---

### Cost vs Reliability Trade-off

```
Redundancy Level    Cost    Reliability    Downtime Risk
────────────────────────────────────────────────────────
None                $       Low           High
Basic (1 standby)   $$      Medium        Medium
Full redundancy     $$$     High          Low
Geographic spread   $$$$    Very High     Very Low
```

#### Considerations

- **Budget**: More redundancy = higher costs
- **Business impact**: What's the cost of downtime?
- **SLA requirements**: What availability did you promise?
- **Industry standards**: Healthcare vs. blog has different needs

---

## Database Concepts

### Database Nodes

**Node**: A single computer/server in a multi-computer setup.

In database context:
- Each node can be a database server
- Nodes work together as a cluster
- Can be distributed geographically

---

### Database Cluster

**Definition**: Multiple database nodes working together as a unified system.

#### Common Setup: Master-Slave Architecture

```
┌─────────────────────────────────────────┐
│         Database Cluster                 │
│                                          │
│     ┌─────────────────┐                 │
│     │  MASTER NODE    │                 │
│     │  (Write Node)   │                 │
│     └────────┬─────── ┘                 │
│              │                           │
│      ┌───────┴───────┐                  │
│      │               │                   │
│  ┌───▼─────┐   ┌────▼─────┐            │
│  │ SLAVE 1 │   │ SLAVE 2  │            │
│  │ (Read)  │   │  (Read)  │            │
│  └─────────┘   └──────────┘            │
│   (Replica)      (Replica)              │
└─────────────────────────────────────────┘
```

#### Roles

**Master/Primary/Write Node**:
- Handles all write operations (INSERT, UPDATE, DELETE)
- Usually only ONE active write node
- Source of truth for data changes

**Slave/Replica/Read Node**:
- Handles read operations (SELECT)
- Receives data from master
- Can have multiple read replicas
- Reduces load on master

---

### Database Replication

**Definition**: Creating and maintaining duplicate copies of data across multiple nodes.

#### Types of Replication

**1. Synchronous Replication** (Consistent)

```
Write to Master
    ↓
Replicate to Slave 1 (wait...)
    ↓
Replicate to Slave 2 (wait...)
    ↓
All confirmed
    ↓
Write CONFIRMED to client

Characteristics:
- Strongly consistent ✓
- Higher latency ✗
- Guaranteed data on all nodes
```

**2. Asynchronous Replication** (Eventually Consistent)

```
Write to Master
    ↓
Immediately CONFIRMED to client ✓
    ↓
Replicate to Slave 1 (background)
    ↓
Replicate to Slave 2 (background)

Characteristics:
- Eventually consistent ⚠️
- Lower latency ✓
- Possible stale reads
```

---

### Read-Heavy vs Write-Heavy Applications

#### Typical Read/Write Ratio

Most applications are **read-heavy**:
- 80% reads (SELECT)
- 20% writes (INSERT, UPDATE, DELETE)

#### Optimization Strategy

```
For Read-Heavy Apps:

Backend ──→ [Write Node] (1 server)
        └──→ [Read Replica 1] (Many servers)
        └──→ [Read Replica 2]
        └──→ [Read Replica 3]
        └──→ [Read Replica N]

Scale reads independently from writes!
```

---

### Latency

**Definition**: The time delay from request to response.

#### Measuring Latency

```
User makes request
    ↓
[  100 milliseconds  ] ← Latency
    ↓
User receives response
```

#### Latency in Database Context

```
Application → Database → Response

Synchronous Replication:
- Higher latency (wait for all nodes)
- Example: 50ms → 200ms

Asynchronous Replication:
- Lower latency (confirm immediately)
- Example: 50ms → 55ms
```

#### Impact on User Experience

- **Low latency**: Fast, responsive application ✓
- **High latency**: Slow, frustrating application ✗
- Trade-off with consistency requirements

---

### Multi-Master Setup

#### Definition

Multiple nodes can accept write operations simultaneously.

```
┌────────────┐     ┌────────────┐
│  Master 1  │◄───→│  Master 2  │
│  (Write)   │     │  (Write)   │
└────────────┘     └────────────┘
      ↕                  ↕
┌────────────┐     ┌────────────┐
│  Replica 1 │     │  Replica 2 │
└────────────┘     └────────────┘
```

#### Challenges

- **Conflict resolution**: What if both masters update the same data?
- **Synchronization**: Keeping masters in sync
- **Complexity**: Much more difficult to implement
- **Consistency**: Harder to maintain

#### When to Consider

- Need for geographical distribution
- Very high write throughput requirements
- Advanced use cases only

#### Typical Approach

For most applications, stick with:
- **One active write node**
- **One standby write node** (for redundancy, not active writes)
- **Multiple read replicas**

---

## ACID Properties

**ACID**: A set of properties that guarantee reliable database transactions.

Most **relational databases** (SQL) are ACID compliant:
- PostgreSQL ✓
- MySQL ✓
- SQL Server ✓
- Oracle ✓

Some NoSQL databases sacrifice some ACID properties for performance/scalability.

---

### A - Atomicity

**Definition**: Transactions are "all or nothing"—they either complete fully or not at all.

#### Example

```
Bank Transfer Transaction:
1. Deduct $100 from Account A
2. Add $100 to Account B

Atomicity ensures:
✓ BOTH steps succeed, OR
✓ BOTH steps fail

✗ Never: Step 1 succeeds, Step 2 fails
   (Money disappears!)
```

#### Key Points

- Transaction = single atomic unit
- No partial completions
- If anything fails, entire transaction rolls back
- Prevents inconsistent states

#### Think of it like

An atom (indivisible unit) - though we can split atoms, the concept is the same: a transaction is the smallest unit of work.

---

### C - Consistency

**Definition**: Database moves from one valid state to another valid state, respecting all defined rules.

⚠️ **Note**: This is different from distributed system consistency!

#### Example: Referential Integrity

```
Database Rules:
- Orders must have valid Customer ID (foreign key)
- Product IDs must exist in Products table

Consistency prevents:
✗ Creating order with non-existent customer
✗ Deleting customer who has orders (without cascade)
✗ Violating unique constraints
✗ Breaking check constraints
```

#### What It Protects

- **Foreign key constraints**
- **Unique constraints**
- **Check constraints**  
- **Data type constraints**
- **Custom business rules**

#### Result

Data has **integrity** = Data is correct and trustworthy

---

### I - Isolation

**Definition**: Concurrent transactions execute independently without interfering with each other.

#### The Problem Without Isolation

```
Transaction 1: Read balance ($100)
Transaction 2: Read balance ($100)
Transaction 1: Deduct $50, Write ($50)
Transaction 2: Deduct $30, Write ($70) ← Should be $20!

Without isolation: Lost update!
```

#### With Proper Isolation

```
Transaction 1: Lock → Read → Modify → Write → Unlock
                                               ↓
Transaction 2: Wait... → Read → Modify → Write

Result: Correct final value
```

#### Isolation Levels (from least to most strict)

1. **Read Uncommitted**: No isolation, dirty reads possible
2. **Read Committed**: Only see committed data
3. **Repeatable Read**: Same reads return same results
4. **Serializable**: Full isolation, like running sequentially

#### Trade-off

- **Higher isolation** = More correctness, less concurrency
- **Lower isolation** = More concurrency, potential anomalies

---

### D - Durability

**Definition**: Once a transaction is committed, it's permanently saved—even if system crashes immediately after.

#### Example

```
User clicks "Save"
    ↓
Transaction commits ✓
    ↓
Power outage 1 millisecond later 💥
    ↓
System restarts
    ↓
Data is still there ✓

Durability guarantees: Committed = Permanent
```

#### How It's Achieved

- Write-ahead logging (WAL)
- Transaction logs
- Disk persistence
- Backup mechanisms
- RAID configurations

#### What It Prevents

✗ Data loss after commit
✗ Rollback after confirmation
✗ Incomplete writes

#### User Trust

Users can trust that when the system confirms an action, it's truly saved.

---

### ACID Summary Table

| Property | Focus | Ensures |
|----------|-------|---------|
| **Atomicity** | Transaction completion | All or nothing |
| **Consistency** | Data rules | Valid state transitions |
| **Isolation** | Concurrency | Independent transactions |
| **Durability** | Persistence | Permanent once committed |

---

## Database Partitioning & Sharding

### Database Partitioning

**Definition**: Splitting a database table into smaller pieces to improve performance and manageability.

Can be done **within a single database instance**.

---

### Vertical Partitioning

**Definition**: Splitting table by columns.

#### When to Use

- Tables with many columns
- Some columns accessed much more frequently
- Large column data (BLOBs, text)

#### Example

**Original Table: Users**
```
┌────┬─────────┬─────────┬──────────┬──────────┬──────────┐
│ ID │  Name   │  Email  │ Address  │   Bio    │  Photo   │
├────┼─────────┼─────────┼──────────┼──────────┼──────────┤
│ 1  │  John   │ j@e.com │  123 St  │  Long... │  Binary..│
│ 2  │  Jane   │ jane@   │  456 Ave │  Long... │  Binary..│
└────┴─────────┴─────────┴──────────┴──────────┴──────────┘
```

**After Vertical Partitioning:**

**users_basic** (frequently accessed):
```
┌────┬─────────┬─────────┐
│ ID │  Name   │  Email  │
├────┼─────────┼─────────┤
│ 1  │  John   │ j@e.com │
│ 2  │  Jane   │ jane@   │
└────┴─────────┴─────────┘
```

**users_details** (rarely accessed):
```
┌────┬──────────┬──────────┬──────────┐
│ ID │ Address  │   Bio    │  Photo   │
├────┼──────────┼──────────┼──────────┤
│ 1  │  123 St  │  Long... │  Binary..│
│ 2  │  456 Ave │  Long... │  Binary..│
└────┴──────────┴──────────┴──────────┘
```

**Join with foreign key when full data needed**

#### Benefits

- Faster queries on frequently accessed columns
- Reduced I/O for common operations
- Better caching efficiency

---

### Horizontal Partitioning

**Definition**: Splitting table by rows (distributing data across multiple tables).

#### When to Use

- Very large tables
- Clear data segmentation (by date, region, category)
- Want to query subsets of data

#### Example

**Original Table: orders** (millions of rows)
```
┌────┬──────────┬────────┬────────────┐
│ ID │ UserID   │ Amount │    Date    │
├────┼──────────┼────────┼────────────┤
│ 1  │  100     │  50.00 │ 2024-01-01 │
│ 2  │  101     │  75.00 │ 2024-01-15 │
│ 3  │  102     │ 120.00 │ 2024-02-01 │
│ 4  │  103     │  30.00 │ 2024-02-10 │
└────┴──────────┴────────┴────────────┘
```

**After Horizontal Partitioning (by month):**

**orders_2024_01**:
```
┌────┬──────────┬────────┬────────────┐
│ 1  │  100     │  50.00 │ 2024-01-01 │
│ 2  │  101     │  75.00 │ 2024-01-15 │
└────┴──────────┴────────┴────────────┘
```

**orders_2024_02**:
```
┌────┬──────────┬────────┬────────────┐
│ 3  │  102     │ 120.00 │ 2024-02-01 │
│ 4  │  103     │  30.00 │ 2024-02-10 │
└────┴──────────┴────────┴────────────┘
```

#### Benefits

- Query only relevant partitions (faster)
- Easier data management (archive old partitions)
- Parallel processing possible
- Reduced index size per partition

#### Database Support

Most modern databases support horizontal partitioning natively (PostgreSQL, MySQL, etc.) and can present a "virtual table" combining all partitions.

---

### Sharding

**Definition**: Horizontal partitioning across **multiple physical database servers** (distributed).

Sharding = Partitioning + Distribution

#### Key Difference

| Partitioning | Sharding |
|-------------|----------|
| Same database instance | Multiple database servers |
| Logical division | Physical division |
| Simpler | More complex |

#### Example: Sharding by User ID

**Original**: One massive database

**Sharded**: Data split across 3 database servers

```
Shard 1 (Server 1):          Shard 2 (Server 2):          Shard 3 (Server 3):
Users 1-1000                 Users 1001-2000              Users 2001-3000
┌──────────────┐            ┌──────────────┐            ┌──────────────┐
│   Database   │            │   Database   │            │   Database   │
│   Shard 1    │            │   Shard 2    │            │   Shard 3    │
└──────────────┘            └──────────────┘            └──────────────┘
```

#### Sharding Strategies

**1. Range-Based Sharding**
```
Shard 1: User IDs 1-1000
Shard 2: User IDs 1001-2000
Shard 3: User IDs 2001-3000
```

**2. Hash-Based Sharding**
```
hash(user_id) % 3 = shard_number
User 1523 → hash → 2 → Shard 2
```

**3. Geographic Sharding**
```
Shard 1: North America users
Shard 2: Europe users
Shard 3: Asia users
```

**4. Directory-Based Sharding**
```
Lookup table maps each key to its shard
More flexible but requires lookup
```

#### Benefits

✅ **Scalability**: Near-infinite horizontal scaling
✅ **Performance**: Each shard handles less data
✅ **Geographic distribution**: Data closer to users
✅ **Fault isolation**: One shard failure doesn't kill all

#### Challenges

❌ **Complexity**: Much harder to implement and maintain
❌ **Cross-shard queries**: Joining data across shards is difficult
❌ **Rebalancing**: Adding/removing shards is complex
❌ **Uneven distribution**: Some shards may get more load

#### When to Consider Sharding

Only when:
- You've exhausted vertical scaling
- You've optimized queries and indexes
- Single database can't handle the load
- You have true "big data" problems

Most applications **don't need sharding**.

---

## Caching Introduction

### What is Caching?

**Definition**: Temporarily storing frequently accessed data in fast-access storage to reduce computation and database load.

### Why Cache?

```
Without Cache:
User Request → Backend → Database Query → Process → Response
                         (Slow: 50-200ms)

With Cache:
User Request → Backend → Check Cache → Response
                         (Fast: 1-5ms)
```

### Benefits

- **Reduced latency**: Much faster response times
- **Lower database load**: Fewer queries to database
- **Better scalability**: Handle more users with same resources
- **Cost savings**: Reduce database server requirements

### What to Cache

- Database query results
- API responses
- Computed values
- User session data
- Static assets (images, CSS, JS)

### Caching Layers

```
┌──────────────┐
│   Browser    │ ← Client-side cache
└──────────────┘
       ↓
┌──────────────┐
│     CDN      │ ← Content Delivery Network
└──────────────┘
       ↓
┌──────────────┐
│   Backend    │ ← Application cache (Redis, Memcached)
└──────────────┘
       ↓
┌──────────────┐
│   Database   │ ← Database query cache
└──────────────┘
```

### Common Caching Technologies

- **Redis**: In-memory data store, supports complex data structures
- **Memcached**: Simple, fast key-value cache
- **CDN**: CloudFlare, AWS CloudFront for static assets
- **Browser**: Built-in HTTP caching

### Cache Invalidation

The challenge: **When to update/remove cached data?**

Strategies:
1. **Time-based expiration** (TTL): Cache expires after X seconds
2. **Event-based**: Clear cache when data changes
3. **Manual**: Developer triggers cache clear
4. **LRU (Least Recently Used)**: Remove oldest unused items

### Impact on System Design

With effective caching, you can often:
- **Delay complex database clustering**
- **Reduce number of database replicas needed**
- **Handle traffic spikes** without scaling
- **Maintain simple architecture** longer

**Key Principle**: Implement caching before complex database setups.

---

## Key Takeaways

### 1. Applications Are Complex Behind the Scenes

What users see as a simple interface involves:
- Multiple servers and databases
- Careful coordination between components
- Trade-offs between consistency, availability, and performance
- Sophisticated error handling and redundancy

### 2. Cloud Services Abstract Complexity

#### Serverless Databases

**"Serverless" = Fully Managed** (servers still exist, you just don't manage them)

Benefits:
- **Elastic**: Automatically scales up and down
- **Pay-per-use**: No over-provisioning waste
- **Hands-free**: No manual clustering/replication
- **High availability**: Built-in redundancy

Examples:
- AWS Aurora Serverless
- Google Cloud Firestore
- Azure Cosmos DB
- PlanetScale

When serverless makes sense:
- ✅ Variable workloads
- ✅ Want to focus on application, not infrastructure
- ✅ Rapid development/prototyping
- ⚠️ Can get expensive at very high scale
- ⚠️ Less control over optimization

### 3. Start Simple, Scale as Needed

#### Recommended Progression

```
Stage 1: Single server (monolith)
    ↓
Stage 2: Three-tier (frontend, backend, database separated)
    ↓
Stage 3: Add caching layer (Redis)
    ↓
Stage 4: Add database read replicas
    ↓
Stage 5: Horizontal scaling with load balancing
    ↓
Stage 6: Advanced (sharding, microservices) - IF NEEDED
```

**Don't jump to Stage 6 immediately!**

### 4. Caching is a Game-Changer

Before implementing complex solutions:
1. **Add caching** (Redis, Memcached)
2. **Optimize queries** (indexes, query analysis)
3. **Use CDN** for static assets
4. **Implement application-level caching**

This can eliminate the need for:
- Complex database clustering
- Multiple database replicas
- Expensive scaling solutions

### 5. Understand Trade-offs

Every decision involves trade-offs:

| Increase | To Get | But Lose |
|----------|--------|----------|
| Availability | Uptime | Cost, Complexity |
| Consistency | Accuracy | Write speed |
| Scalability | Capacity | Simplicity |
| Redundancy | Reliability | Cost |
| Elasticity | Efficiency | Setup complexity |

**There's no perfect solution**—choose based on your requirements.

### 6. Know Your Requirements

Before making decisions, ask:
- What availability do we need? (How many nines?)
- Can we tolerate eventual consistency?
- What's our read/write ratio?
- What's our growth projection?
- What's our budget?
- What's the cost of downtime?

### 7. Most Apps Don't Need the Most Complex Solutions

Reality check:
- ❌ Most apps don't need 6-nines availability
- ❌ Most apps don't need sharding
- ❌ Most apps don't need multi-master databases
- ✅ Most apps work great with simple 3-tier + caching
- ✅ Most apps benefit more from optimization than scaling

**Start simple, scale when you have real data showing you need to.**

---

## Next Steps

### Continue Learning

Topics to explore next:
1. **Scaling applications** from 1 to millions of users
2. **Load balancing** strategies and algorithms
3. **Caching** in depth (strategies, technologies, patterns)
4. **Database optimization** (indexing, query optimization)
5. **System design interviews** (real-world problems)
6. **Message queues** and event-driven architecture
7. **Microservices** architecture patterns

### Practice

- Design systems for real-world scenarios
- Set up simple distributed systems locally
- Experiment with Redis/Memcached
- Try cloud platforms (AWS, GCP, Azure)
- Work through system design interview questions

### Resources

- **Backend Engineering Mind Map**: Comprehensive technology guide
- **Mentorship Programs**: Get one-on-one help
- **System Design Primer**: GitHub repository
- **Cloud Provider Documentation**: AWS, Google Cloud, Azure

---

## Glossary

### Core Terms

- **Availability**: System uptime and accessibility
- **Reliability**: System correctness and functionality
- **Consistency**: All users see same data simultaneously
- **Latency**: Time from request to response
- **Scalability**: Ability to handle increased load
- **Elasticity**: Automatic resource adjustment

### Architecture Terms

- **Node**: Single server in a distributed system
- **Cluster**: Multiple nodes working together
- **Replica**: Duplicate copy of data/server
- **Shard**: Partition of data across servers
- **SPOF**: Single Point of Failure
- **Redundancy**: Backup components for fault tolerance

### Database Terms

- **ACID**: Atomicity, Consistency, Isolation, Durability
- **Master/Primary**: Main database accepting writes
- **Slave/Replica**: Database copy for reads
- **Replication**: Copying data across nodes
- **Partitioning**: Splitting data within database
- **Sharding**: Splitting data across databases

### Scaling Terms

- **Vertical Scaling**: Adding resources to one server
- **Horizontal Scaling**: Adding more servers
- **Scale Up**: Vertical scaling synonym
- **Scale Out**: Horizontal scaling synonym
- **Load Balancing**: Distributing traffic across servers

### Consistency Terms

- **Strong Consistency**: Immediate consistency across all nodes
- **Eventual Consistency**: Eventual consistency, temporary inconsistency allowed
- **Synchronous**: Wait for completion before proceeding
- **Asynchronous**: Don't wait, continue immediately

---

## Final Thoughts

System design is about **making informed trade-offs** based on your specific requirements. There's no one-size-fits-all solution.

**Key Principles**:
1. Start simple
2. Measure before optimizing
3. Scale based on real needs, not hypothetical ones
4. Understand your constraints and requirements
5. Learn from others but adapt to your context

The concepts covered here form the foundation for understanding modern distributed systems. As you continue learning, these principles will help you make better architectural decisions.

**Remember**: Even simple architectures can serve millions of users if designed well and optimized properly. Don't over-engineer!

---

*This guide provides foundational knowledge in system design. Continue learning, practicing, and building to deepen your understanding of these concepts.*