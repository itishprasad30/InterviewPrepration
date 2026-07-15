# System Design Interview Prep — 100 Questions (3-5 YOE Level)

Concise What/Why/How format. Grouped by topic so you can drill weak areas. Last section has full "design X" case-study questions, which is where interviewers actually combine everything below.

---

## Section 1: Fundamentals & Scalability Concepts (1-10)

**1. What is Scalability? Vertical vs Horizontal scaling?**
Scalability = system's ability to handle growing load. **Vertical (scale up)** — bigger machine (more CPU/RAM), simple but has a hard ceiling and single point of failure. **Horizontal (scale out)** — more machines, near-limitless but needs the app to be stateless/distributable.

**2. What is Latency vs Throughput?**
Latency = time for one request to complete. Throughput = number of requests handled per unit time. They're related but not the same — you can have low latency and low throughput, or higher latency but massive throughput via parallelism.

**3. What is Availability? How is "five nines" defined?**
Percentage of time a system is operational. 99.999% ("five nines") ≈ ~5 minutes downtime/year. Achieved via redundancy — no single point of failure.

**4. CAP Theorem — what and why does it matter?**
In a distributed system, during a network partition you can only guarantee **2 of 3**: Consistency, Availability, Partition tolerance. Since partitions are unavoidable in real distributed systems, the real choice is CP (consistent but may reject requests) vs AP (available but may serve stale data).

**5. What is Consistency (strong vs eventual)?**
**Strong consistency** — every read gets the latest write immediately (e.g., traditional RDBMS). **Eventual consistency** — replicas converge to the same value *eventually*, reads may briefly see stale data (common in NoSQL/distributed caches) — trades correctness for availability/speed.

**6. What is a Single Point of Failure (SPOF) and how do you eliminate it?**
Any component whose failure takes down the whole system. Eliminated via redundancy — multiple instances behind a load balancer, DB replicas, multi-AZ/region deployment.

**7. What is Fault Tolerance vs High Availability?**
Fault tolerance = system keeps working correctly *despite* component failures (no visible disruption). High availability = system stays *up* and reachable, may have brief hiccups but recovers fast (failover). Fault tolerance is stricter/costlier.

**8. What are the back-of-envelope estimation basics interviewers expect?**
Rough numbers you should know: 1 day ≈ 86,400s. Read-heavy systems are often 100:1 read:write. 1 KB record × 1M users ≈ 1GB. Being able to estimate QPS, storage, and bandwidth roughly (not precisely) shows practical judgment.

**9. What is a Monolith vs Microservices — trade-offs?**
Monolith — single deployable unit, simpler to develop/deploy/debug early on, but scales and deploys as one block. Microservices — independently deployable/scalable services (like your Humine setup — payroll, leave, notifications), better scalability/team autonomy, but adds network overhead, distributed debugging complexity, and eventual consistency challenges.

**10. What is Statelessness and why does it matter for scaling?**
A stateless service doesn't store session/request-specific data locally between requests — any instance can handle any request. Critical for horizontal scaling behind a load balancer (no "sticky" requirement tying a user to one server).

---

## Section 2: Load Balancing (11-18)

**11. What is a Load Balancer and why use one?**
Distributes incoming traffic across multiple server instances — improves availability (routes around failed instances) and scalability (spreads load).

**12. Layer 4 vs Layer 7 Load Balancing?**
**L4** — operates at TCP/UDP level, routes based on IP/port, fast but "blind" to content. **L7** — operates at HTTP level, can route based on URL path, headers, cookies — more flexible, slightly more overhead.

**13. Common Load Balancing algorithms?**
Round robin (equal rotation), least connections (send to least-busy server), weighted round robin (accounts for server capacity differences), IP hash (consistent routing per client — useful when you need session stickiness).

**14. What is Sticky Session and when would you need it?**
Routes a given user's requests to the *same* backend server consistently — needed if session state is stored in-memory on that server (not ideal — better to externalize session state to Redis and stay fully stateless).

**15. How does a Load Balancer detect a failed server?**
Health checks — periodic pings/HTTP calls to a `/health` endpoint; unhealthy instances are automatically removed from the rotation.

**16. What is DNS-based load balancing?**
DNS returns multiple IPs (or rotates them) for a domain, spreading load at the DNS resolution level — coarse-grained, doesn't react quickly to failures compared to a dedicated LB.

**17. What is a reverse proxy and how is it different from a load balancer?**
Reverse proxy sits in front of servers, forwarding client requests to the appropriate backend (can also do caching, SSL termination, compression). Load balancing is *one* function a reverse proxy can perform, but reverse proxies do more (e.g., Nginx as both).

**18. What is Global Server Load Balancing (GSLB)?**
Routes users to the nearest/healthiest **data center/region** (not just server) — typically DNS-based, considers geographic proximity and regional health.

---

## Section 3: Caching (19-30)

**19. What is Caching and why use it?**
Storing frequently accessed data in a fast-access layer (memory) to avoid recomputing/re-fetching from a slower source (DB/disk/network) — dramatically reduces latency and DB load.

**20. Where can caching happen? (layers)**
Client-side (browser cache), CDN (edge caching for static assets), application-level (in-memory, e.g., Caffeine), distributed cache (Redis/Memcached), DB query cache.

**21. Redis vs Memcached — key differences?**
Redis — supports rich data structures (lists, sets, sorted sets, hashes), persistence, pub/sub, replication. Memcached — simpler, purely key-value, multi-threaded, slightly faster for pure simple caching. Redis is generally preferred today for its versatility.

**22. What are cache eviction policies?**
**LRU** (Least Recently Used — evict what hasn't been accessed recently), **LFU** (Least Frequently Used), **FIFO**, **TTL-based** (expire after a fixed time). LRU is the most common default.

**23. What is Cache-Aside (Lazy Loading) pattern?**
App checks cache first; on a miss, reads from DB, then populates the cache for next time. Simple, cache only holds what's actually requested, but first request always has a miss penalty.
```
if (cache.has(key)) return cache.get(key);
data = db.query(key);
cache.set(key, data);
return data;
```

**24. What is Write-Through caching?**
Every write goes to the cache **and** the DB synchronously, together. Keeps cache always consistent with DB, but adds write latency (waits for both).

**25. What is Write-Behind (Write-Back) caching?**
Write goes to cache immediately, then asynchronously flushed to DB later. Fast writes, but risk of data loss if the cache crashes before flushing.

**26. What is Cache Stampede (Thundering Herd) and how do you prevent it?**
When a popular cache key expires, many concurrent requests simultaneously hit the DB to repopulate it, overwhelming it. Prevented via: locking (only one request repopulates, others wait), staggered/jittered TTLs, or serving stale data while refreshing in the background.

**27. What is Cache Invalidation and why is it "one of the two hard problems in CS"?**
Ensuring stale cached data gets removed/updated when the underlying data changes. Hard because you must track *all* the ways data can change and *all* the places it's cached, without over-invalidating (killing cache benefit) or under-invalidating (serving stale data).

**28. TTL-based expiry vs event-based invalidation — trade-offs?**
TTL — simple, but you're always trading off "freshness window" vs cache hit rate. Event-based (invalidate on write) — always fresh but requires reliable event propagation (e.g., pub/sub) and more coordination complexity.

**29. What is a CDN (Content Delivery Network) and how does it reduce latency?**
A geographically distributed network of edge servers caching static (and sometimes dynamic) content close to users — reduces round-trip distance, offloads origin servers.

**30. How would you cache a frequently-changing leaderboard/ranking?**
Use a Redis Sorted Set (`ZSET`) — O(log n) insert/update, O(log n) range queries for top-N, avoids recomputing ranking from the DB on every request.

---

## Section 4: Database Scaling & Design (31-50)

**31. Vertical vs Horizontal DB scaling?**
Vertical — bigger DB server (more RAM/CPU/faster disk), simple but limited and costly at the high end. Horizontal — spreading data across multiple DB servers (sharding/partitioning), scales further but adds complexity (cross-shard queries, joins).

**32. What is Database Replication and Master-Slave (Primary-Replica) setup?**
Copies of data maintained across multiple DB servers. Primary handles writes, replicas handle reads (read scaling) and serve as failover. Replication can be synchronous (consistent, slower) or asynchronous (faster, replicas can lag).

**33. What is Read Replica lag and why does it matter?**
Delay between a write on the primary and it appearing on a replica (with async replication). Matters because a user might write data then immediately read it from a lagging replica and see stale/missing data ("read-your-writes" problem).

**34. What is Database Sharding and how do you choose a shard key?**
Splitting data horizontally across multiple independent DB instances, each holding a subset. Shard key should distribute load evenly (avoid hotspots) and align with your most common query pattern (e.g., shard by `employee_id` if most queries filter by employee).

**35. What are common sharding strategies?**
**Range-based** (shard by value ranges — simple but can create hotspots), **Hash-based** (hash the key — even distribution, but range queries become hard), **Directory-based** (lookup service maps key → shard — flexible but adds a dependency/SPOF risk).

**36. What is the downside of sharding?**
Cross-shard joins/transactions become very hard, resharding (adding/removing shards) is operationally painful, and you lose some of the simplicity/ACID guarantees a single DB gives you.

**37. What is Database Partitioning (within a single DB)?**
Splitting one large table into smaller physical pieces (by range, hash, or list of values) **within the same database instance** — improves query performance and manageability without the cross-server complexity of sharding.

**38. SQL vs NoSQL — when do you pick each?**
SQL — structured data, complex relationships/joins, strong consistency/ACID needed (e.g., payroll transactions). NoSQL — flexible/evolving schema, massive horizontal scale, eventual consistency acceptable (e.g., activity logs, session data, product catalogs).

**39. Types of NoSQL databases — name a few and their use case.**
**Key-Value** (Redis, DynamoDB — simple lookups). **Document** (MongoDB — flexible JSON-like docs). **Column-family** (Cassandra — huge write throughput, time-series). **Graph** (Neo4j — relationship-heavy data like social networks).

**40. What is Database Indexing and why does it speed up reads?**
A separate sorted structure (typically B-tree) pointing to row locations — turns an O(n) full table scan into an O(log n) lookup for indexed columns. Trade-off: extra storage + slower writes (index must update too).

**41. When would an index hurt performance?**
Heavy-write tables (every insert/update also updates the index), low-cardinality columns (e.g., boolean flags — index doesn't help narrow results much), or too many indexes on one table bloating write cost.

**42. What is a Composite Index and does column order matter?**
An index on multiple columns together. Order matters — the index is efficiently usable only for queries filtering on a **leading prefix** of the indexed columns (index on `(dept_id, salary)` helps `WHERE dept_id=X` and `WHERE dept_id=X AND salary>Y`, but not `WHERE salary>Y` alone).

**43. What is Database Connection Pooling and why is it important?**
Reuses a fixed pool of DB connections instead of opening/closing a new one per request — connection setup is expensive (TCP handshake, auth), pooling avoids that overhead and prevents overwhelming the DB with too many concurrent connections (e.g., HikariCP in Spring Boot).

**44. What is Denormalization and when would you deliberately do it?**
Intentionally duplicating data to avoid expensive joins, trading storage/write complexity for faster reads — common in read-heavy systems (e.g., storing an employee's name directly on a payroll record instead of always joining).

**45. What is Optimistic vs Pessimistic Locking?**
**Pessimistic** — lock the row on read, preventing others from modifying until you're done (safe but hurts concurrency). **Optimistic** — no lock; check a version number on write, reject if it's changed since you read it (better concurrency, needs a retry strategy on conflict).

**46. How would you handle a "hot" DB row that's updated very frequently (e.g., a global counter)?**
Avoid hitting the DB row directly for every increment — batch updates, use an in-memory counter (Redis `INCR`, atomic) that periodically syncs to the DB, or shard the counter into multiple sub-counters summed on read.

**47. What is Database Failover and how does it typically work?**
When the primary DB fails, a replica is promoted to primary automatically (or via orchestration tooling) to minimize downtime — requires health monitoring + a consensus mechanism to avoid "split-brain" (two nodes both thinking they're primary).

**48. What is a Write-Ahead Log (WAL) and why do DBs use it?**
Changes are written to an append-only log *before* being applied to the actual data files — ensures durability/crash recovery (DB can replay the log to recover state after a crash) and underpins replication in many DBs.

**49. How do you design a schema for multi-tenancy (e.g., serving multiple companies like Decathlon from one platform)?**
Options: **Separate DB per tenant** (strong isolation, costly at scale), **Shared DB, separate schema per tenant** (middle ground), **Shared DB and tables with a `tenant_id` column** (most scalable, needs careful query scoping/row-level security to avoid cross-tenant leaks).

**50. What is Database Migration strategy for zero-downtime schema changes?**
Backward-compatible incremental changes — e.g., add a new column as nullable first, deploy code that writes to both old/new, backfill data, then switch reads to new column, finally drop the old one. Avoids a single risky "big bang" migration.

---

## Section 5: Rate Limiting (51-58)

**51. What is Rate Limiting and why is it needed?**
Restricting how many requests a client can make in a given time window — protects against abuse, ensures fair usage, prevents a single client from overwhelming the system (directly relevant to your notification service work).

**52. What is the Token Bucket algorithm?**
A bucket holds tokens, refilled at a fixed rate up to a max capacity; each request consumes a token, rejected if the bucket's empty. Allows short bursts (up to bucket size) while enforcing an average rate over time.

**53. What is the Leaky Bucket algorithm?**
Requests enter a queue (bucket) and are processed at a fixed constant rate, regardless of burst size — smooths out traffic, but doesn't allow bursts the way token bucket does.

**54. What is Fixed Window Counter rate limiting and its flaw?**
Counts requests in fixed time windows (e.g., per minute), resets each window. Flaw: a burst right at the boundary of two windows (e.g., end of minute 1 + start of minute 2) can let through nearly 2x the intended limit.

**55. What is Sliding Window Log / Sliding Window Counter, and how does it fix the fixed-window flaw?**
Sliding window tracks requests over a rolling time frame (not fixed boundaries) — either by logging exact timestamps (log, accurate but memory-heavy) or by weighting the current + previous window counts proportionally (counter, approximate but efficient). Avoids the boundary-burst problem.

**56. Where do you implement rate limiting — API Gateway, app layer, or DB layer?**
Ideally as early as possible — API Gateway/reverse proxy level (rejects excess traffic before it even hits your services, saving resources). App-layer rate limiting (like your tiered notification limiter) is used when limits are business-logic-specific (e.g., per-user daily notification caps), not just generic traffic shaping.

**57. How do you rate-limit across multiple server instances (distributed rate limiting)?**
Can't rely on in-memory counters per instance (each would have its own count). Use a shared store like Redis with atomic operations (`INCR` + `EXPIRE`), so all instances see the same counter.

**58. What HTTP status code and headers should a rate-limited response include?**
`429 Too Many Requests`, with headers like `Retry-After` (seconds until retry is allowed) and often `X-RateLimit-Limit`/`X-RateLimit-Remaining` to help clients self-throttle.

---

## Section 6: Messaging, Async & Event-Driven Systems (59-68)

**59. Why use a Message Queue between services instead of direct synchronous calls?**
Decouples producer and consumer — producer doesn't need the consumer to be available/fast; smooths traffic spikes (buffer); enables retry/failure isolation without cascading failures across services.

**60. What is the difference between a Message Queue and a Pub/Sub system?**
**Queue** — one message consumed by exactly one consumer (work distribution, e.g., processing payroll jobs). **Pub/Sub** — one message delivered to *all* subscribed consumers (event broadcast, e.g., notifying multiple services when an employee is onboarded).

**61. Kafka vs RabbitMQ — when would you pick each?**
**Kafka** — high-throughput, durable log-based, great for event streaming/replay, many consumers reading independently at their own pace. **RabbitMQ** — traditional message broker, more complex routing (exchanges/queues), good for task queues with lower throughput needs. Kafka scales further for event-driven architectures.

**62. What is "at-least-once" vs "at-most-once" vs "exactly-once" delivery?**
**At-most-once** — message might be lost, never duplicated. **At-least-once** — message guaranteed delivered but might be duplicated (consumer must handle duplicates/be idempotent). **Exactly-once** — hardest to guarantee, usually achieved via idempotency + deduplication rather than true magic.

**63. Why must consumers be idempotent in an at-least-once delivery system?**
Because the same message can be redelivered (e.g., after a consumer crash before acking) — if processing it twice has side effects (like double-charging or double-sending a notification), you need idempotency keys/checks to avoid duplicate effects — same pattern as your double-submit bug fix.

**64. What is a Dead Letter Queue (DLQ)?**
A separate queue where messages that repeatedly fail processing get routed after N retries — prevents a "poison message" from blocking the queue indefinitely, and lets you inspect/reprocess failures manually.

**65. What is Event Sourcing?**
Storing state changes as an immutable sequence of events (rather than just the current state) — current state is derived by replaying events. Gives a full audit trail and the ability to reconstruct past states, at the cost of added complexity.

**66. What is the Outbox Pattern and what problem does it solve?**
Solves the problem of atomically updating a DB **and** publishing an event (two separate systems can't share a transaction). Instead, you write the event to an "outbox" table in the *same* DB transaction as your business data change, then a separate process reliably publishes it from the outbox — avoiding lost/inconsistent events.

**67. What is backpressure in a streaming/queue system?**
When a consumer can't keep up with the producer's rate — backpressure mechanisms (buffering, rate-limiting producers, or explicit flow-control signals) prevent the consumer from being overwhelmed/crashing.

**68. How would you design the notification service you built to handle bulk sends reliably?**
Queue-based: API enqueues notification jobs instead of sending synchronously; workers pull from the queue and send asynchronously with retry/backoff on failure, tiered rate limits enforced (as you built) using a shared counter (Redis), and a DLQ for permanently failed sends.

---

## Section 7: API Design & Communication (69-78)

**69. REST vs gRPC vs GraphQL — trade-offs?**
**REST** — simple, widely understood, resource-based, over-fetching/under-fetching can be an issue. **gRPC** — binary protocol (Protobuf), very fast, great for internal service-to-service calls, less browser-friendly. **GraphQL** — client specifies exactly what fields it needs (avoids over-fetching), but adds complexity on the server (query resolution, caching is harder).

**70. What is API Versioning and common strategies?**
Managing breaking changes without disrupting existing clients — via URL path (`/v1/employees`), header-based, or query param. URL-based is most common/discoverable.

**71. What is an API Gateway and what does it typically handle?**
Single entry point for client requests routed to appropriate microservices — commonly handles auth, rate limiting, request routing, logging/monitoring, and sometimes request/response transformation — keeps these cross-cutting concerns out of individual services.

**72. Synchronous vs Asynchronous communication between microservices — when to use which?**
**Sync (REST/gRPC)** — needed when the caller needs an immediate response to proceed (e.g., checking leave balance before approving). **Async (message queue)** — when the caller doesn't need to wait, or when decoupling/resilience matters more than immediacy (e.g., sending a notification after leave approval).

**73. What is Idempotency in API design and why does it matter for `POST` vs `PUT`?**
An idempotent operation produces the same result no matter how many times it's repeated. `PUT`/`DELETE` are expected to be idempotent by HTTP convention; `POST` is not — which is exactly why retries of `POST` (e.g., form resubmission) can cause duplicates unless you add an explicit idempotency key, like your double-submit fix.

**74. What is Pagination and why does offset-based pagination struggle at scale?**
Splitting large result sets into pages. `OFFSET`-based pagination gets slower on deep pages because the DB still scans/skips all preceding rows. **Cursor/keyset pagination** (`WHERE id > last_seen_id`) avoids this by jumping directly via an indexed column.

**75. What is Circuit Breaker pattern and why use it between services?**
Monitors calls to a downstream service; if failures exceed a threshold, it "trips" and stops sending requests for a cooldown period (failing fast instead of waiting on a struggling/dead service) — prevents cascading failures across a microservices chain.

**76. What is Retry with Exponential Backoff, and why not just retry immediately?**
Retrying a failed call with increasing delays between attempts (often with jitter/randomness added) — prevents a herd of retries from hammering an already-struggling service right after it fails (which can make recovery worse, not better).

**77. What is a Bulkhead pattern?**
Isolating resources (thread pools, connections) per downstream dependency so that if one dependency slows down/fails, it doesn't exhaust shared resources and take down calls to *other* unrelated dependencies too.

**78. How would you design an API to handle a client uploading a large file (e.g., an employee document)?**
Options: direct upload to the API with streaming (avoid loading whole file in memory), or (more scalable) have the API generate a pre-signed URL for direct upload to object storage (S3), avoiding routing large payloads through your app servers at all.

---

## Section 8: Storage, Search & Data Formats (79-86)

**79. Object Storage (S3-like) vs Block Storage vs File Storage — differences?**
**Object storage** — flat namespace, stores blobs with metadata, accessed via API (great for documents/images, e.g., your S3-based policy PDFs). **Block storage** — raw disk blocks, used for VM/DB storage, lower-level. **File storage** — traditional hierarchical filesystem, shared via NFS/SMB.

**80. Why use Elasticsearch alongside a relational DB instead of relying on SQL `LIKE` search?**
Elasticsearch is built for full-text search — inverted indexes make text search fast and relevance-ranked, supports fuzzy matching, filters, aggregations at scale that SQL `LIKE '%term%'` can't do efficiently (`LIKE` with a leading wildcard can't use a standard B-tree index at all).

**81. What is an Inverted Index (used by search engines)?**
Maps each word/term to the list of documents containing it (the reverse of a normal document → words mapping) — enables fast lookup of "which documents contain X" instead of scanning every document.

**82. JSON vs Protocol Buffers (Protobuf) — trade-offs?**
JSON — human-readable, ubiquitous, larger payload, slower to parse. Protobuf — binary, compact, schema-enforced, much faster to serialize/deserialize — preferred for internal high-throughput service-to-service communication (commonly paired with gRPC).

**83. What is Data Replication Consistency trade-off (sync vs async) in storage systems?**
Synchronous replication — write isn't acknowledged until all replicas confirm (strong consistency, higher write latency). Asynchronous — write acknowledged immediately, replicas catch up shortly after (lower latency, risk of data loss if primary fails before replicating).

**84. How would you design storage for time-series data (e.g., attendance/step-count logs)?**
Favor a column-family/time-series-optimized store (Cassandra, InfluxDB, TimescaleDB) — partition by time range, since queries are typically range-based ("last 30 days") and writes are append-heavy/sequential, which these stores are optimized for over a general RDBMS.

**85. What is Data Archiving/Cold Storage and why separate it from hot data?**
Moving rarely-accessed old data (e.g., payroll records older than 3 years) to cheaper, slower storage — keeps your primary hot-path DB smaller/faster and reduces cost, since hot storage (SSD-backed, indexed) is expensive relative to cold storage (e.g., S3 Glacier).

**86. What is a Blob and why not store large files directly in a relational DB column?**
Storing large binary files (images, PDFs) directly in DB rows bloats the DB, slows backups/replication, and wastes an expensive resource on something object storage handles better/cheaper — best practice is to store the file in object storage and just keep a reference URL/key in the DB row.

---

## Section 9: Security & Reliability at Scale (87-94)

**87. How do you secure service-to-service communication in a microservices architecture?**
mTLS (mutual TLS — both sides verify each other's certificate), or service-mesh-level authentication (e.g., Istio), plus short-lived service tokens/API keys rather than static shared secrets.

**88. What is Encryption at Rest vs In Transit?**
**At rest** — data encrypted while stored on disk (protects against physical/storage breach). **In transit** — data encrypted while moving over the network (TLS/HTTPS, protects against interception). Both are typically required for compliance with sensitive data like payroll/PII.

**89. How would you design an audit logging system for sensitive actions (e.g., salary changes)?**
Append-only log (never update/delete), capturing who/what/when/old-value/new-value, ideally written to a separate store from the main transactional DB (so it survives even if the primary is compromised), with restricted write access.

**90. What is Blue-Green Deployment and why use it?**
Two identical production environments ("blue" = live, "green" = new version); traffic is switched over once the new version is verified — enables instant rollback (just switch back) and zero-downtime deploys.

**91. What is Canary Deployment?**
Gradually rolling out a new version to a small subset of traffic/users first, monitoring for errors, then expanding — limits blast radius of a bad deploy compared to releasing to 100% of traffic at once.

**92. What is Chaos Engineering (briefly)?**
Deliberately injecting failures (killing instances, adding latency) into a system in a controlled way to verify it actually handles failures gracefully, rather than assuming it does.

**93. How do you monitor a distributed system effectively?**
The "three pillars": **Metrics** (aggregated numeric data — request rate, error rate, latency — often via Prometheus/Grafana), **Logs** (detailed per-event records), **Traces** (following a single request across multiple services — critical in microservices to pinpoint where latency/errors originate).

**94. What is Distributed Tracing and why is it hard without it in microservices?**
Attaches a unique trace ID to a request that propagates across every service it touches, letting you reconstruct the full call chain and see where time was spent/where it failed. Without it, debugging a slow/failed request across many independently-deployed services is like finding a needle in a haystack across separate logs.

---

## Section 10: "Design X" Case-Study Questions (95-100)

These are where interviewers combine everything above. Structure your answer as: **clarify requirements → estimate scale → high-level architecture → deep dive on 1-2 hard parts → discuss trade-offs.**

**95. Design a URL Shortener (e.g., bit.ly).**
Key points: base62 encoding or hash+collision-check for short codes, DB choice (key-value store fine, since access pattern is simple lookup), caching hot URLs (Redis), analytics via async event logging, custom alias support, expiry handling.

**96. Design a Rate Limiter for an API Gateway.**
Key points: choose algorithm (token bucket for bursts, sliding window for accuracy), where to store counters for distributed consistency (Redis with atomic ops), per-user vs per-IP vs per-API-key limits, returning `429` + `Retry-After`.

**97. Design a Notification System (push/email/SMS) — directly relevant to your Humine work.**
Key points: queue-based architecture (decouple trigger from send), worker pool consuming the queue, tiered rate limiting per user (daily/hourly/minute caps — as you built), retry with backoff + DLQ for failures, template management, delivery status tracking, idempotency to avoid duplicate sends on retry.

**98. Design a Leave/Attendance Management System — directly relevant to your domain.**
Key points: data model (employee, leave type, leave balance, leave request with status workflow), handling concurrent leave requests against the same balance (optimistic locking or DB-level constraints to prevent overdraw), approval workflow with notifications, LOP (loss of pay) calculation as an async/batch job at month-end, audit trail for HR compliance.

**99. Design a File Storage/Document Sharing System (e.g., for employee documents/salary slips).**
Key points: object storage backend (S3) with the DB storing only metadata/references, pre-signed URLs for secure time-limited access (avoids the IDOR pattern you fixed — never serve by raw guessable ID without an ownership check), versioning, virus scanning on upload, access control per document.

**100. Design a Distributed Cache (like a mini-Redis).**
Key points: hash-based sharding across nodes (consistent hashing to minimize re-shuffling when nodes are added/removed), replication per shard for fault tolerance, eviction policy (LRU), TTL support, handling node failure (rebalancing), client-side vs server-side sharding decision.

---

## How to actually use this list
- Don't just memorize definitions — for every topic, be ready to say **why** it exists (what problem it solves) and **one real trade-off**. That's what separates a 3-5 YOE answer from a fresher's answer.
- The "Design X" questions (95-100) are where the interview actually happens — everything above is vocabulary you need *in service of* those conversations. Practice narrating your thought process out loud for at least Q96-99, since they map closely to systems you've actually worked on (notification service, leave system, document storage).
- Lean hard on real Humine stories: rate limiting (Q51-58, 96-97), idempotency (Q73, 62-63), IDOR/security (Q87-89, 99) — concrete experience beats textbook answers every time.

Want a deep-dive walkthrough of one of the "Design X" questions (95-100) as a full mock system design interview, where I push back and ask follow-ups the way a real interviewer would?
