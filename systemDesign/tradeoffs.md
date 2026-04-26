# System Design Trade-offs

A reference guide to the most common architectural trade-offs in distributed systems. Every decision has a cost — the goal is to pick the right trade-off for your specific requirements, not find a universally correct answer.

---

## Table of Contents

1. [SQL vs NoSQL](#1-sql-vs-nosql)
2. [Vertical Scaling vs Horizontal Scaling](#2-vertical-scaling-vs-horizontal-scaling)
3. [Monolith vs Microservices](#3-monolith-vs-microservices)
4. [REST vs GraphQL vs gRPC](#4-rest-vs-graphql-vs-grpc)
5. [Consistency vs Availability (CAP Theorem)](#5-consistency-vs-availability-cap-theorem)
6. [Strong Consistency vs Eventual Consistency](#6-strong-consistency-vs-eventual-consistency)
7. [Synchronous vs Asynchronous Communication](#7-synchronous-vs-asynchronous-communication)
8. [Push vs Pull Architecture](#8-push-vs-pull-architecture)
9. [Caching vs Fresh Data](#9-caching-vs-fresh-data)
10. [Normalization vs Denormalization](#10-normalization-vs-denormalization)
11. [Batch Processing vs Stream Processing](#11-batch-processing-vs-stream-processing)
12. [Read Replicas vs Write Scaling](#12-read-replicas-vs-write-scaling)
13. [Fan-out on Write vs Fan-out on Read](#13-fan-out-on-write-vs-fan-out-on-read)
14. [Latency vs Throughput](#14-latency-vs-throughput)
15. [Storage Cost vs Query Performance](#15-storage-cost-vs-query-performance)

---

## 1. SQL vs NoSQL

### SQL (Relational Databases)

Stores data in structured tables with a fixed schema and enforces relationships via foreign keys. Supports ACID transactions natively.

**Examples:** PostgreSQL, MySQL, Aurora, CockroachDB

```
┌─────────────┐         ┌──────────────┐
│   orders    │         │    users     │
├─────────────┤         ├──────────────┤
│ id          │────────>│ id           │
│ user_id(FK) │         │ name         │
│ total       │         │ email        │
│ created_at  │         └──────────────┘
└─────────────┘
```

| Strengths | Weaknesses |
|-----------|-----------|
| ACID transactions (financial-grade consistency) | Harder to scale horizontally (sharding is complex) |
| Rich querying (JOINs, aggregations, window functions) | Schema changes require migrations (ALTER TABLE) |
| Referential integrity enforced by DB | Performance degrades on very high write throughput |
| Mature tooling and ecosystem | Rigid schema — poor fit for polymorphic or sparse data |

### NoSQL (Non-Relational Databases)

Flexible schema, horizontally scalable, optimized for specific access patterns. Different subtypes for different needs.

| Type | Examples | Best For |
|------|---------|---------|
| **Key-Value** | Redis, DynamoDB | Session store, cache, simple lookups by ID |
| **Document** | MongoDB, Firestore | Semi-structured data (JSON), flexible schema per record |
| **Wide-Column** | Cassandra, HBase | Time-series, write-heavy, large-scale event logs |
| **Graph** | Neo4j, Amazon Neptune | Highly connected data (social graphs, fraud detection) |

| Strengths | Weaknesses |
|-----------|-----------|
| Horizontal scale built-in | No native JOINs — application must join in code |
| Flexible / schema-less (easy to iterate) | Eventual consistency by default (depends on config) |
| Optimized for specific access patterns | Fewer guarantees on transactions (varies by DB) |
| High write throughput (Cassandra, DynamoDB) | Aggregation queries are often expensive or unsupported |

### When to Choose

```
Use SQL when:                          Use NoSQL when:
✓ Data is relational (JOINs needed)   ✓ Massive scale, high write throughput
✓ ACID transactions are required      ✓ Flexible / evolving schema
✓ Complex queries and reporting       ✓ Simple access patterns (lookup by key)
✓ Team is familiar with relational    ✓ Data is naturally hierarchical (JSON)
  model                               ✓ Horizontal sharding is a hard requirement
```

**Real-world split:** Orders DB → PostgreSQL (ACID). Product catalog → DynamoDB (flexible schema, high read throughput). Session store → Redis (key-value, sub-millisecond).

---

## 2. Vertical Scaling vs Horizontal Scaling

### Vertical Scaling (Scale Up)

Add more resources (CPU, RAM, disk) to an existing server.

```
Before:          After:
┌──────────┐     ┌──────────────┐
│ 8 CPU    │ →   │ 64 CPU       │
│ 16 GB    │     │ 256 GB RAM   │
│ 500 GB   │     │ 4 TB SSD     │
└──────────┘     └──────────────┘
```

| Strengths | Weaknesses |
|-----------|-----------|
| Simple — no application changes needed | Hard ceiling (largest machine available) |
| No distributed system complexity | Single point of failure |
| Works well for stateful systems (SQL) | Expensive at the high end (non-linear cost) |
| Low operational overhead | Requires downtime to resize in many cases |

### Horizontal Scaling (Scale Out)

Add more servers and distribute load across them.

```
              ┌─────────────┐
              │ Load        │
              │ Balancer    │
              └──────┬──────┘
          ┌──────────┼──────────┐
     ┌────▼───┐ ┌────▼───┐ ┌───▼────┐
     │ Server │ │ Server │ │ Server │
     │   1    │ │   2    │ │   3    │
     └────────┘ └────────┘ └────────┘
```

| Strengths | Weaknesses |
|-----------|-----------|
| Virtually unlimited scale | Application must be stateless (or use shared session store) |
| No single point of failure | Distributed system complexity (consistency, coordination) |
| Linear cost scaling | Network latency between nodes |
| Rolling deploys with zero downtime | Load balancer becomes a bottleneck if not designed well |

### When to Choose

```
Vertical scaling:                    Horizontal scaling:
✓ Early stage — simplicity matters   ✓ Traffic grows unpredictably
✓ Stateful workloads (primary DB)    ✓ Need fault tolerance / high availability
✓ Quick fix for immediate load spike ✓ Stateless services (web servers, APIs)
✓ Budget allows high-end hardware    ✓ Cloud-native, auto-scaling environments
```

**Common pattern:** Scale vertically first (simple), then horizontally once the vertical ceiling is hit. Most systems use both — vertical for stateful DB primaries, horizontal for stateless application servers.

---

## 3. Monolith vs Microservices

### Monolith

All application components are packaged and deployed as a single unit.

```
┌──────────────────────────────────────┐
│            Monolith                  │
│  ┌─────────┐  ┌─────────┐           │
│  │  Auth   │  │ Orders  │           │
│  └─────────┘  └─────────┘           │
│  ┌─────────┐  ┌─────────┐           │
│  │ Catalog │  │ Payments│           │
│  └─────────┘  └─────────┘           │
└──────────────────────────────────────┘
         Single deployable unit
```

| Strengths | Weaknesses |
|-----------|-----------|
| Simple local development (one repo, one process) | Scaling — must scale the whole app, not just hot components |
| No network latency for inter-module calls | One bad deploy takes down everything |
| Easy to trace, debug, and test end-to-end | Large codebase becomes hard to navigate |
| Transactions span modules natively | Technology lock-in — entire app shares one stack |

### Microservices

Application is decomposed into small, independently deployable services that communicate over the network.

```
 ┌──────────┐   ┌──────────┐   ┌──────────┐
 │  Auth    │   │  Orders  │   │ Catalog  │
 │ Service  │   │ Service  │   │ Service  │
 └────┬─────┘   └────┬─────┘   └────┬─────┘
      │              │              │
      └──────────────┴──────────────┘
              Message Bus / API Gateway
```

| Strengths | Weaknesses |
|-----------|-----------|
| Independent scaling per service | Distributed system complexity (network, latency, partial failure) |
| Independent deploys — faster release cycles | Cross-service transactions are hard (sagas, 2PC) |
| Technology heterogeneity (right tool per service) | Operational overhead (K8s, service mesh, observability) |
| Fault isolation — one service crashes, others keep running | Debugging across services requires distributed tracing |

### When to Choose

```
Monolith first when:                  Microservices when:
✓ Early stage, small team             ✓ Team > 10+ engineers (Conway's Law)
✓ Domain boundaries are unclear       ✓ Services have wildly different scale needs
✓ Speed of iteration matters most     ✓ Independent deployment velocity is critical
✓ Operational complexity is a burden  ✓ Domain boundaries are well understood
```

**The rule of thumb:** Start monolith, extract services when a clear scaling or team boundary emerges. Premature microservices add enormous complexity with little benefit.

---

## 4. REST vs GraphQL vs gRPC

### Comparison

| Dimension | REST | GraphQL | gRPC |
|-----------|------|---------|------|
| **Protocol** | HTTP/1.1 or HTTP/2 | HTTP/1.1 or HTTP/2 | HTTP/2 |
| **Data format** | JSON / XML | JSON | Protobuf (binary) |
| **Schema** | OpenAPI (optional) | Strongly typed SDL | Strongly typed `.proto` |
| **Fetching** | Fixed response shape per endpoint | Client specifies exact fields | Fixed RPC methods |
| **Over-fetching** | Common | Eliminated | No (explicit methods) |
| **Streaming** | Limited (SSE / WebSocket add-on) | Subscriptions (WebSocket) | Native bi-directional streaming |
| **Performance** | Good | Good (less bandwidth) | Excellent (binary, HTTP/2 multiplexing) |
| **Browser support** | Native | Native | Requires grpc-web proxy |
| **Best for** | Public APIs, simple CRUD | Complex data graphs, mobile clients | Internal service-to-service, high performance |

### When to Choose

```
REST:                         GraphQL:                    gRPC:
✓ Public-facing API           ✓ Mobile clients (save       ✓ Internal microservice
✓ Simple CRUD operations        bandwidth)                   communication
✓ Wide ecosystem support      ✓ Complex, nested data       ✓ Performance-critical paths
✓ Caching via HTTP            ✓ Rapidly changing UI        ✓ Streaming (chat, telemetry)
✓ Easy to understand            requirements               ✓ Polyglot services with
  by external consumers       ✓ BFF (Backend for             strong contracts
                                Frontend) pattern
```

---

## 5. Consistency vs Availability (CAP Theorem)

The CAP theorem states that a distributed system can guarantee only **two of three** properties simultaneously:

```
           Consistency
               /\
              /  \
             /    \
            /  CA  \
           /        \
          /----CP-AP-\
    Availability — Partition Tolerance
```

| Type | Guarantees | Sacrifices | Examples |
|------|-----------|-----------|---------|
| **CP** (Consistent + Partition Tolerant) | Every read returns the latest write | Availability during partition | HBase, ZooKeeper, etcd |
| **AP** (Available + Partition Tolerant) | System responds even during partition | May return stale data | Cassandra, CouchDB, DynamoDB (eventual) |
| **CA** (Consistent + Available) | Strong consistency + always available | Cannot tolerate network partition | Single-node RDBMS (not realistic in distributed systems) |

**Key insight:** Network partitions are unavoidable in real distributed systems. The real choice is **CP vs AP** — do you prefer to reject requests (CP) or return possibly stale data (AP) when a partition occurs?

```
Banking system → CP: reject transactions rather than risk inconsistency
Social media feed → AP: show slightly stale feed rather than return an error
```

---

## 6. Strong Consistency vs Eventual Consistency

### Strong Consistency

Every read reflects the most recent write, globally. All nodes agree on the current state before returning.

```
Write "X=5" to Node A
  → Synchronously replicate to Node B and Node C
    → Only return success when all replicas confirm
      → Any subsequent read from any node returns X=5
```

| Strengths | Weaknesses |
|-----------|-----------|
| Simplest mental model — no stale reads | High write latency (must wait for all replicas) |
| Required for financial / inventory data | Lower availability (node failure blocks writes) |
| No conflict resolution needed | Reduced throughput under high write load |

### Eventual Consistency

Writes propagate asynchronously. Nodes may temporarily diverge but will converge to the same state given no new writes.

```
Write "X=5" to Node A
  → Return success immediately
    → Replicate to Node B and C asynchronously (milliseconds later)
      → Read from Node B immediately after: may return X=3 (old value)
        → Read from Node B after replication: returns X=5
```

| Strengths | Weaknesses |
|-----------|-----------|
| Low write latency (no wait for replica ack) | Stale reads are possible |
| High availability — writes succeed even if replicas are down | Conflict resolution complexity (last-write-wins, CRDTs) |
| High throughput | Application must handle stale data gracefully |

### When to Choose

```
Strong Consistency:                   Eventual Consistency:
✓ Bank balance, account debit         ✓ Social media like count
✓ Inventory reservation               ✓ Product view counter
✓ Seat booking / ticket purchase      ✓ User profile updates
✓ Any operation where stale data      ✓ DNS records
  causes real harm                    ✓ Shopping cart (with conflict merge)
```

---

## 7. Synchronous vs Asynchronous Communication

### Synchronous

Caller sends a request and blocks until the response is received.

```
Order Service ──── POST /charge ────> Payment Service
             <──── 200 OK (card charged) ────
     (waits the entire time)
```

| Strengths | Weaknesses |
|-----------|-----------|
| Simple request-response model | Caller is blocked — latency adds up in chains |
| Immediate feedback (success / failure) | Tight coupling — callee downtime = caller failure |
| Easy to debug and trace | Cascading failures (one slow service stalls all upstream) |

### Asynchronous

Caller publishes a message and continues. A consumer processes it independently.

```
Order Service ──── publish "order.placed" ────> Kafka
                                                  │
                                    ┌─────────────┴──────────────┐
                                    ▼                            ▼
                             Payment Service           Fulfillment Service
                             (processes async)         (processes async)
```

| Strengths | Weaknesses |
|-----------|-----------|
| Decoupled — services evolve independently | No immediate feedback (must poll or use callbacks) |
| High throughput — no blocking | Harder to debug (distributed tracing needed) |
| Absorbs traffic spikes (queue as buffer) | Message ordering and deduplication complexity |
| Resilient — producer works even if consumer is down | At-least-once delivery may require idempotency |

### When to Choose

```
Synchronous:                          Asynchronous:
✓ User-facing request needs           ✓ Background jobs (email, transcoding)
  immediate result                    ✓ Decoupling independent services
✓ Simple, low-volume workflows        ✓ Absorbing traffic spikes
✓ Transaction spans multiple          ✓ Fan-out to many consumers
  services atomically                 ✓ Retry and dead-letter queue patterns
```

---

## 8. Push vs Pull Architecture

### Push

Server proactively sends data to clients or downstream systems when something changes.

```
New tweet posted
  → Fan-out Service pushes tweet ID into each follower's timeline cache
    → All 1M followers have the tweet ready before they even open the app
```

| Strengths | Weaknesses |
|-----------|-----------|
| Low read latency — data pre-delivered | High write amplification (1 write → N pushes) |
| Real-time delivery | Wasted work if recipient never reads (inactive users) |
| Good for small, active audiences | Celebrity problem — 100M followers = 100M writes |

### Pull

Clients or downstream systems request data when they need it.

```
User opens feed
  → Timeline Service fetches latest tweets from each followed account
    → Merge + sort → return
```

| Strengths | Weaknesses |
|-----------|-----------|
| No wasted work — only computed when needed | Higher read latency (computed at request time) |
| Works well for large / celebrity followings | High read amplification under concurrent load |
| Simple write path | Can overload origin on thundering herd |

### When to Choose

```
Push:                                 Pull:
✓ Small active audience               ✓ Large or unknown audience size
✓ Real-time delivery critical         ✓ Infrequent reads (batch)
✓ Regular users (social feeds)        ✓ Celebrity accounts (high follower count)
✓ Notification systems                ✓ Polling for status updates
```

**Hybrid (X/Twitter's approach):** Push for regular users (< 20K followers), pull for celebrities at read time — merge at timeline assembly.

---

## 9. Caching vs Fresh Data

### Caching

Store a copy of frequently read data in fast memory (Redis, Memcached, CDN) to avoid repeated expensive lookups.

```
Request for user profile
  → Check Redis → HIT → return in < 1ms
  → Check Redis → MISS → query PostgreSQL (10ms) → write to Redis → return
```

| Strengths | Weaknesses |
|-----------|-----------|
| Sub-millisecond read latency | Stale data — cache may not reflect latest DB state |
| Reduces DB load dramatically | Cache invalidation is notoriously hard |
| Absorbs read traffic spikes | Memory is finite — eviction causes cache misses |
| CDN caching eliminates origin server load | Cold start — empty cache on new deploy |

### Cache Invalidation Strategies

| Strategy | How | Trade-off |
|----------|-----|-----------|
| **TTL (Time-to-Live)** | Cache expires after N seconds | Simple; stale data for up to TTL duration |
| **Write-through** | Update cache and DB simultaneously on write | Always fresh; double write cost |
| **Write-behind** | Write to cache first, async to DB | Low write latency; risk of data loss |
| **Cache-aside** | App reads cache; on miss, reads DB + writes cache | Most common; miss penalty on first read |
| **Event-driven invalidation** | DB change event triggers cache delete | Near-fresh; complex to implement |

### When to Cache

```
Cache aggressively:                   Avoid caching:
✓ Static or slow-changing data        ✗ Rapidly changing data (stock prices)
✓ High read-to-write ratio            ✗ User-specific, sensitive data (be careful)
✓ Expensive computations              ✗ Data where stale reads cause real harm
✓ Session data, tokens, preferences     (inventory, bank balance)
✓ CDN: images, CSS, JS
```

---

## 10. Normalization vs Denormalization

### Normalization

Eliminate data redundancy by splitting data into separate tables linked by foreign keys.

```
Normalized:
┌──────────┐    ┌──────────┐    ┌──────────┐
│  orders  │    │  items   │    │ products │
│ order_id │───>│ order_id │    │  prod_id │
│ user_id  │    │ prod_id  │───>│  name    │
│ total    │    │ qty      │    │  price   │
└──────────┘    └──────────┘    └──────────┘
Query requires 3-way JOIN to show order line items with product names
```

| Strengths | Weaknesses |
|-----------|-----------|
| No data duplication — single source of truth | JOINs are expensive at scale |
| Easy to update — change data in one place | Complex queries for reads |
| Strong data integrity | Poor read performance on large datasets |

### Denormalization

Intentionally duplicate data to optimize read performance — embed related data directly.

```
Denormalized (document or wide-column):
┌─────────────────────────────────────────────┐
│ order_id: 123                               │
│ user_name: "Sirisha"  ← copied from users   │
│ items: [                                    │
│   { product_name: "Laptop", qty: 1,         │
│     price: 999 }   ← copied from products  │
│ ]                                           │
└─────────────────────────────────────────────┘
Read is a single document fetch — no JOIN
```

| Strengths | Weaknesses |
|-----------|-----------|
| Fast reads — no JOINs required | Data duplication — updates must touch multiple places |
| Works well with document and wide-column stores | Risk of inconsistency if update is partial |
| Predictable, constant-time lookups | More storage consumed |

### When to Choose

```
Normalize when:                       Denormalize when:
✓ Write-heavy workloads               ✓ Read-heavy workloads
✓ Data changes frequently             ✓ Data changes infrequently
✓ OLTP systems (banking, CRM)         ✓ OLAP / analytics systems
✓ Consistency is paramount            ✓ Low latency reads are critical
                                      ✓ NoSQL document stores
```

---

## 11. Batch Processing vs Stream Processing

### Batch Processing

Accumulate data over a period of time, then process it all at once as a job.

```
Raw event logs → stored in S3 for 24 hours
  → Spark job runs at midnight
    → Reads all events → computes daily active users, revenue totals
      → Writes results to data warehouse
```

**Examples:** Apache Spark, Hadoop MapReduce, AWS Glue, dbt

| Strengths | Weaknesses |
|-----------|-----------|
| High throughput — optimized for large datasets | High latency — results only available after job completes |
| Simpler to develop and test | Not suitable for real-time use cases |
| Cheaper — can run on spot/preemptible instances | Data is stale by the time it's processed |
| Reprocessing is easy — replay from raw data | Large jobs can fail partway through |

### Stream Processing

Process each event as it arrives, in real time.

```
User clicks "like" → Kafka event
  → Flink job reads event immediately
    → Increments like counter in Redis
      → Updates trending score
        → Result visible in < 1 second
```

**Examples:** Apache Flink, Apache Kafka Streams, Apache Spark Structured Streaming, AWS Kinesis

| Strengths | Weaknesses |
|-----------|-----------|
| Low latency — near-real-time results | More complex to develop (state management, watermarks) |
| Continuous processing — no waiting for a job window | Higher infrastructure cost (always running) |
| Stateful aggregations over time windows | Exactly-once semantics are hard to achieve |
| Triggers real-time alerts and actions | Reprocessing historical data requires separate batch job |

### When to Choose

```
Batch Processing:                     Stream Processing:
✓ Daily/weekly reports                ✓ Real-time dashboards
✓ ML model retraining                 ✓ Fraud detection
✓ ETL pipelines to data warehouse     ✓ Live leaderboards
✓ Full dataset reprocessing           ✓ Trending topic detection
✓ Historical analytics                ✓ Real-time alerting and monitoring
```

**Lambda Architecture:** Run both in parallel — stream processing for real-time approximate results, batch processing for accurate historical results, merged at query time.

---

## 12. Read Replicas vs Write Scaling

### Read Replicas

Add replicas that serve read traffic, while all writes go to the primary.

```
         Writes
           │
    ┌──────▼──────┐
    │   Primary   │
    │    (RW)     │
    └──────┬──────┘
           │ async replication
    ┌──────┴──────┬──────────────┐
    ▼             ▼              ▼
┌────────┐  ┌────────┐   ┌────────┐
│Replica1│  │Replica2│   │Replica3│
│  (RO)  │  │  (RO)  │   │  (RO)  │
└────────┘  └────────┘   └────────┘
       ← Read traffic distributed →
```

| Strengths | Weaknesses |
|-----------|-----------|
| Scales read throughput linearly | Replication lag — replicas may serve stale data |
| Offloads analytics queries from primary | Writes still bottleneck on single primary |
| Geographic distribution of reads | Read-your-own-write requires routing to primary |
| Simple to add without schema changes | Replica lag monitoring required |

### Write Scaling (Sharding)

Partition data horizontally across multiple primary nodes — each shard owns a subset of the data.

```
┌─────────────────────────────────────────────┐
│ Shard Router: user_id % 3                   │
└──────┬──────────────┬──────────────┬────────┘
       │              │              │
┌──────▼──┐     ┌─────▼───┐   ┌─────▼───┐
│ Shard 0 │     │ Shard 1 │   │ Shard 2 │
│users    │     │ users   │   │ users   │
│0,3,6... │     │ 1,4,7.. │   │ 2,5,8.. │
└─────────┘     └─────────┘   └─────────┘
```

| Strengths | Weaknesses |
|-----------|-----------|
| Scales write throughput horizontally | Cross-shard queries / JOINs are expensive |
| Each shard is smaller → faster queries | Rebalancing shards is operationally complex |
| Fault isolation — one shard failure is partial | Hot shards (uneven distribution) degrade performance |

### When to Choose

```
Read Replicas:                        Sharding:
✓ Read-heavy workload (10:1 ratio)    ✓ Write-heavy workload
✓ Analytics queries slow down primary ✓ Single node can't hold all the data
✓ Geographic read distribution        ✓ Write throughput exceeds primary capacity
✓ Simple to implement first           ✓ Accepted operational complexity
```

---

## 13. Fan-out on Write vs Fan-out on Read

Covered in depth in the X (Twitter) design — summarized here as a general trade-off.

### Fan-out on Write (Push)

When a user posts, immediately deliver the content to all followers' precomputed feeds.

| Strengths | Weaknesses |
|-----------|-----------|
| O(1) read — feed is ready and waiting | O(N) write — one post triggers N cache writes |
| Low read latency | Wasted computation for inactive followers |
| Predictable read performance | Celebrity problem (100M followers = 100M writes) |

### Fan-out on Read (Pull)

When a user loads their feed, fetch and merge posts from all followed accounts at read time.

| Strengths | Weaknesses |
|-----------|-----------|
| O(1) write — post stored once | O(N) read — must query N followed accounts |
| No wasted work for inactive users | High read latency for users following many accounts |
| Simple write path | Read amplification under high concurrency |

**Hybrid (recommended):** Push for regular users, pull for high-follower accounts. Merge at read time.

---

## 14. Latency vs Throughput

### Latency

Time for a single request to complete (P50, P95, P99).

### Throughput

Number of requests the system can handle per unit of time (requests/sec, messages/sec).

**The tension:** Optimizing for one often hurts the other.

```
Batching increases throughput, increases latency:
  → Wait to accumulate 100 messages before writing to DB
    → Single DB write (high throughput)
    → But message 1 waits for messages 2–100 before being written (high latency)

Processing immediately decreases latency, decreases throughput:
  → Write every message to DB as it arrives
    → Message 1 is persisted immediately (low latency)
    → 100 DB writes instead of 1 (lower throughput)
```

| Optimization | Latency | Throughput | Technique |
|-------------|---------|-----------|-----------|
| Batching writes | Worse | Better | Kafka producer batch, DB bulk insert |
| Async processing | Better (for caller) | Better | Message queues, background workers |
| Caching | Better | Better (reduces DB load) | Redis, CDN |
| Connection pooling | Better | Better | PgBouncer, HikariCP |
| Compression | Slightly worse | Better (less bandwidth) | gzip, Snappy, LZ4 |

### When to Prioritize

```
Prioritize Latency:                   Prioritize Throughput:
✓ User-facing read paths              ✓ Data pipelines and ETL
✓ OTP / authentication flows          ✓ Log aggregation
✓ Payment processing                  ✓ Batch analytics
✓ Real-time gaming                    ✓ Message queues
✓ Trading systems                     ✓ File uploads / downloads
```

---

## 15. Storage Cost vs Query Performance

### Optimized for Storage (Normalized, Compressed)

Minimize data duplication and apply compression — lowest storage cost, higher query cost.

```
Raw event: { user_id: 123, action: "purchase", product_id: 456, timestamp: ... }
  → Stored in Parquet with Snappy compression
  → 10 GB of raw JSON → 1.2 GB compressed Parquet
  → Query requires columnar scan + decompression
```

### Optimized for Query Performance (Denormalized, Indexed, Pre-aggregated)

Store pre-computed results and indexes to make queries fast — higher storage cost, lower query cost.

```
Pre-aggregated daily active user count per region:
  → Stored in Redis: { "US:2026-04-26": 4200000, "EU:2026-04-26": 1800000 }
  → Query: O(1) Redis GET — returns instantly
  → Storage: duplicates and pre-computes data that could be derived from raw events
```

| Technique | Storage Cost | Query Performance |
|-----------|-------------|------------------|
| Raw normalized tables | Low | Slow (JOINs, full scans) |
| Columnar format (Parquet) | Very Low | Good for analytics (column pruning) |
| Secondary indexes | Medium | Fast point lookups |
| Materialized views | Medium–High | Fast — pre-computed |
| Denormalized documents | High | Very fast (single fetch) |
| Pre-aggregated counters (Redis) | High | Extremely fast (O(1)) |

### When to Choose

```
Optimize for storage cost:            Optimize for query performance:
✓ Historical / archival data          ✓ User-facing, latency-sensitive reads
✓ Data accessed infrequently          ✓ High-frequency queries on same data
✓ Cold storage (S3 Glacier)           ✓ Dashboards and real-time analytics
✓ Compliance / audit logs             ✓ Leaderboards, trending counts
✓ Raw event store (data lake)         ✓ Search indexes
```

---

## Quick Reference Summary

| Trade-off | Choose Left When | Choose Right When |
|-----------|-----------------|------------------|
| **SQL vs NoSQL** | Relational data, ACID required | Massive scale, flexible schema |
| **Vertical vs Horizontal Scaling** | Stateful, early-stage | Stateless, high availability needed |
| **Monolith vs Microservices** | Small team, unclear domain boundaries | Large team, independent scaling needed |
| **REST vs gRPC** | Public API, browser clients | Internal services, high performance |
| **Strong vs Eventual Consistency** | Financial data, inventory | Social feeds, counters, analytics |
| **Sync vs Async** | Immediate response needed | Decoupling, background jobs, fan-out |
| **Push vs Pull** | Small active audience, real-time | Large audience, celebrities, batch |
| **Cache vs Fresh Data** | Read-heavy, slow-changing | Write-heavy, must-be-fresh |
| **Normalized vs Denormalized** | Write-heavy, data changes often | Read-heavy, low latency required |
| **Batch vs Stream** | Historical analytics, ETL | Real-time alerts, live dashboards |
| **Read Replicas vs Sharding** | Read-heavy workload | Write-heavy, data outgrows one node |
| **Fan-out on Write vs Read** | Regular users, small following | Celebrities, large inactive user base |
| **Latency vs Throughput** | User-facing, interactive | Pipelines, background processing |
| **Storage vs Query Performance** | Archive, cold data | Hot path, frequent reads |
