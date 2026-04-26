# System Design Concepts

---

## 1. Load Balancing

**Description:** Distributes incoming network traffic across multiple servers to ensure no single server becomes a bottleneck. Can operate at Layer 4 (transport) or Layer 7 (application).

**Algorithms:** Round Robin, Least Connections, IP Hash, Weighted Round Robin

**Use Cases:**
- High-traffic web applications (e.g., Netflix, Amazon) that need horizontal scaling
- Ensuring high availability — if one server fails, traffic reroutes automatically
- Distributing API requests across microservices

---

## 2. Caching

**Description:** Stores frequently accessed data in fast-access storage (memory) to reduce latency and database load. Can exist at multiple layers: client, CDN, application, database.

**Tools:** Redis, Memcached, CDN (CloudFront, Fastly)

**Strategies:** Cache-aside, Write-through, Write-back, Read-through

**Use Cases:**
- Session storage for authenticated users
- Caching database query results to reduce read load
- Storing rendered HTML or API responses at the edge (CDN)

---

## 3. Database Sharding

**Description:** Horizontally partitions a database by splitting data across multiple database instances (shards), each responsible for a subset of data.

**Sharding Keys:** User ID, geographic region, hash-based

**Use Cases:**
- Social networks storing billions of user records (e.g., Facebook)
- Multi-tenant SaaS platforms isolating customer data
- E-commerce platforms with massive product catalogs

---

## 4. Replication

**Description:** Copies data from a primary (master) database to one or more secondary (replica) databases to improve read throughput and fault tolerance.

**Types:** Synchronous (strong consistency), Asynchronous (eventual consistency), Multi-master

**Use Cases:**
- Read-heavy applications that offload reads to replicas
- Disaster recovery — replicas in different availability zones
- Analytics workloads running heavy queries on read replicas without impacting production

---

## 5. Message Queues

**Description:** Enables asynchronous communication between services by placing messages in a queue. Producers write messages; consumers process them independently.

**Tools:** Kafka, RabbitMQ, AWS SQS, Google Pub/Sub

**Use Cases:**
- Decoupling microservices (e.g., order service → payment service → notification service)
- Rate limiting background jobs (email sending, video encoding)
- Event streaming and audit logs

---

## 6. Content Delivery Network (CDN)

**Description:** A geographically distributed network of proxy servers that cache static and dynamic content close to end users to reduce latency.

**Tools:** Cloudflare, AWS CloudFront, Fastly, Akamai

**Use Cases:**
- Serving static assets (images, JS, CSS) globally with low latency
- Video streaming platforms (YouTube, Netflix) delivering content by region
- DDoS mitigation by absorbing traffic at the edge

---

## 7. API Gateway

**Description:** A single entry point for all client requests to backend microservices. Handles routing, authentication, rate limiting, logging, and request transformation.

**Tools:** AWS API Gateway, Kong, NGINX, Apigee

**Use Cases:**
- Aggregating multiple microservice calls into a single client response
- Enforcing authentication (OAuth, JWT) before requests reach services
- Rate limiting public APIs to prevent abuse

---

## 8. Consistent Hashing

**Description:** A hashing technique used in distributed systems to map data to nodes such that adding or removing nodes minimizes data redistribution.

**Use Cases:**
- Distributed caches (Memcached, Redis Cluster) — minimizes cache misses when nodes change
- Distributed databases routing requests to the correct shard
- Load balancers distributing sessions with sticky routing

---

## 9. CAP Theorem

**Description:** States that a distributed system can guarantee only two of three properties simultaneously: **Consistency** (all nodes see the same data), **Availability** (every request gets a response), **Partition Tolerance** (system works despite network splits).

**Real-world trade-offs:**
- CP systems: HBase, Zookeeper (consistent but may reject requests during partitions)
- AP systems: Cassandra, CouchDB (available but may return stale data)
- CA systems: Traditional RDBMS (not partition-tolerant — unsuitable for distributed systems)

**Use Cases:**
- Choosing the right database based on business requirements (banking needs CP; social feeds tolerate AP)

---

## 10. Microservices Architecture

**Description:** Structures an application as a collection of small, independently deployable services, each responsible for a specific business capability and communicating over APIs or message queues.

**Use Cases:**
- Large engineering teams working on isolated domains without deployment coupling
- Services with different scaling needs (e.g., search scales independently from checkout)
- Independent deployment and technology choices per service

---

## 11. Rate Limiting

**Description:** Controls the rate of requests a client can make to an API or service within a time window to prevent abuse, ensure fair use, and protect backend resources.

**Algorithms:** Token Bucket, Leaky Bucket, Fixed Window, Sliding Window

**Use Cases:**
- Public APIs (GitHub, Twitter) limiting requests per user per minute
- Login endpoints to prevent brute-force attacks
- Preventing a single tenant from monopolizing shared infrastructure

---

## 12. Service Discovery

**Description:** Mechanism by which microservices dynamically find the network locations of other services, since instances may start, stop, or move in a cloud environment.

**Types:** Client-side (Eureka), Server-side (AWS ALB, Consul)

**Use Cases:**
- Kubernetes environments where pod IPs change on restarts
- Auto-scaled services registering and deregistering dynamically
- Multi-region deployments routing to the nearest healthy instance

---

## 13. Event-Driven Architecture

**Description:** System design pattern where services communicate by producing and consuming events rather than direct synchronous calls. Promotes loose coupling and scalability.

**Use Cases:**
- E-commerce order pipelines (order placed → inventory reserved → payment charged → email sent)
- Real-time dashboards reacting to a stream of sensor or user-activity events
- Audit logging where all state changes are captured as immutable events

---

## 14. Indexing

**Description:** Data structures (B-Tree, Hash, Inverted Index) built on database columns to speed up query lookups at the cost of additional storage and slower writes.

**Types:** Primary, Secondary, Composite, Full-text, Partial

**Use Cases:**
- Speeding up user lookup by email or username in auth flows
- Full-text search indexes in document stores (Elasticsearch)
- Composite indexes optimizing multi-column filter queries in analytics

---

## 15. Distributed Transactions (SAGA Pattern)

**Description:** Manages data consistency across multiple microservices without a two-phase commit. A saga is a sequence of local transactions; each step publishes an event or triggers a compensating transaction on failure.

**Types:** Choreography (event-driven), Orchestration (central coordinator)

**Use Cases:**
- E-commerce checkout spanning inventory, payment, and fulfillment services
- Travel booking systems coordinating hotel, flight, and car rental reservations
- Financial transfers across accounts in separate services

---

## 16. Blob / Object Storage

**Description:** Stores unstructured binary data (images, videos, backups) as objects with metadata and a unique key, optimized for high durability and large-scale retrieval.

**Tools:** AWS S3, Google Cloud Storage, Azure Blob Storage

**Use Cases:**
- Storing user-uploaded media (profile pictures, documents)
- Static website hosting and CDN origin
- Data lake storage for big data analytics pipelines

---

## 17. Search Systems

**Description:** Dedicated search engines that maintain an inverted index over documents to support full-text search, ranking, filtering, and faceted navigation with low latency.

**Tools:** Elasticsearch, OpenSearch, Solr, Typesense

**Use Cases:**
- Product search on e-commerce sites with filters and relevance ranking
- Log aggregation and search (ELK stack)
- Autocomplete and typeahead suggestions

---

## 18. Heartbeat & Health Checks

**Description:** Periodic signals sent by services to a coordinator (or received by a load balancer) to confirm liveness and readiness. Failed checks trigger failover or restarts.

**Use Cases:**
- Kubernetes liveness and readiness probes restarting unhealthy pods
- Load balancers removing unresponsive instances from the pool
- Distributed lock managers detecting dead lease holders

---

## 19. Circuit Breaker Pattern

**Description:** Monitors calls to a downstream service and "opens" the circuit (stops sending requests) when failures exceed a threshold, allowing the system to fail fast and recover gracefully.

**Tools:** Hystrix, Resilience4j, Istio

**Use Cases:**
- Preventing cascading failures when a payment gateway is degraded
- Protecting databases during traffic spikes by shedding load quickly
- Giving a struggling downstream service time to recover

---

## 20. Horizontal vs. Vertical Scaling

**Description:**
- **Vertical Scaling (Scale Up):** Adding more CPU/RAM to an existing machine. Simple but has hardware limits.
- **Horizontal Scaling (Scale Out):** Adding more machines to a pool. Requires stateless services or distributed state management; effectively unlimited.

**Use Cases:**
- Vertical: Small databases or monoliths where re-architecting isn't feasible short-term
- Horizontal: Stateless web/API servers behind a load balancer, auto-scaling groups in cloud environments
