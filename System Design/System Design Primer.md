# System Design Interview Guide

> **Senior Software Engineer Interview Preparation**  
> Trade-offs, failure modes, and when to choose each approach

---

## Table of Contents

1. [API Gateway](#1-api-gateway)
2. [Load Balancer](#2-load-balancer)
3. [Reverse Proxy](#3-reverse-proxy)
4. [Rate Limiting](#4-rate-limiting)
5. [Throttling](#5-throttling)
6. [Pagination](#6-pagination)
7. [Cache Stampede](#7-cache-stampede)
8. [Idempotency](#8-idempotency)
9. [GraphQL](#9-graphql)
10. [gRPC](#10-grpc)
11. [Webhooks](#11-webhooks)
12. [JWT](#12-jwt)
13. [OAuth 2.0](#13-oauth-20)
14. [Cache Invalidation](#14-cache-invalidation)
15. [Composite Index](#15-composite-index)
16. [Query Optimization](#16-query-optimization)
17. [CAP Theorem](#17-cap-theorem)
18. [ACID](#18-acid)
19. [Sharding](#19-sharding)
20. [Circuit Breaker](#20-circuit-breaker)
21. [Livelock](#21-livelock)
22. [CSRF](#22-csrf)
23. [Backpressure](#23-backpressure)
24. [False Sharing](#24-false-sharing)
25. [mTLS](#25-mtls)

---

## 1. API Gateway

A single entry point that sits in front of your microservices. It handles cross-cutting concerns — **authentication, rate limiting, request routing, protocol translation, response aggregation**. Instead of clients calling 10 services directly, they hit one gateway (e.g., Kong, AWS API Gateway). In practice, it decouples client contracts from internal service topology, enabling independent service evolution.

### Trade-offs
- **Centralization vs. single point of failure** — It simplifies client communication but if the gateway goes down, everything goes down. You must deploy it HA (multi-AZ, auto-scaling groups).
- **Latency overhead** — Every request adds a hop. At ultra-low-latency (HFT, gaming), you may skip it and let clients call services directly with service mesh.
- **Monolithic gateway vs. BFF (Backend-for-Frontend)** — A single gateway for all clients becomes a deployment bottleneck. Netflix moved to federated gateways per domain team. The BFF pattern (one gateway per client type — mobile, web, internal) avoids one team's change breaking another.

### Real-world failure modes
- **Gateway becomes a deployment bottleneck** — every team needs routing changes, config becomes massive.
- **Cascading failures** — If the gateway doesn't have timeouts/circuit breakers for downstream services, one slow service can exhaust gateway threads, killing all routes.
- **Misconfigured auth** — A single misconfigured route in the gateway can expose internal services publicly.

### When to choose
- Use an API Gateway when you have **multiple client types and 5+ microservices**. Skip it for monoliths or very small service counts where a simple reverse proxy suffices.

---

## 2. Load Balancer

Distributes incoming traffic across a pool of backend servers to maximize throughput, minimize latency, and ensure no single node is overwhelmed. Algorithms include **Round Robin, Least Connections, Weighted, Consistent Hashing, and IP Hash**. Operates at **L4 (TCP/UDP)** or **L7 (HTTP — can inspect headers/paths)**. L7 is more flexible (content-based routing) but more expensive. In large systems, you often see a multi-tier setup: DNS-level → L4 (e.g., AWS NLB) → L7 (e.g., Envoy/NGINX).

### Trade-offs
- **L4 vs. L7** — L4 is faster (no payload inspection), but L7 gives content-based routing, sticky sessions, and header manipulation. Use L4 for raw TCP services (databases, gRPC), L7 for HTTP APIs.
- **Least Connections vs. Round Robin** — Round Robin is simple but ignores server load. Least Connections is smarter but requires health tracking. For heterogeneous backends, use **Weighted Least Connections**.
- **Client-side vs. server-side LB** — In microservices, client-side LB (Envoy sidecar, gRPC built-in) avoids the extra hop and single point of failure. Trade-off: every client needs the service registry and LB logic.

### Real-world failure modes
- **Thundering herd on failover** — If one backend dies and the LB shifts all traffic to remaining nodes, they can cascade-fail. Solution: slow-start (ramp new/recovered nodes gradually).
- **Health check misconfiguration** — Shallow health checks (`/health` returns 200 but DB is down) cause the LB to route to broken nodes.
- **Sticky sessions + scaling** — Session affinity prevents even distribution and makes auto-scaling less effective.

### When to choose
- **Always** — any multi-node deployment needs one. The question is *where* and *what layer*. For internal mesh: client-side (Envoy). For edge: managed L7 (ALB, Cloud Load Balancer).

---

## 3. Reverse Proxy

Sits between clients and backend servers, forwarding client requests to the appropriate server. Unlike a forward proxy (which acts on behalf of clients), a reverse proxy acts on behalf of servers. It provides **SSL termination, caching, compression, and anonymity of backend infrastructure**. NGINX and HAProxy are classic examples. Every API gateway is a reverse proxy, but not every reverse proxy is an API gateway.

### Trade-offs
- **Caching at the proxy** — Can massively reduce backend load for static/semi-static content. But cache invalidation becomes your problem.
- **SSL termination** — Offloads crypto from backends (cheaper, simpler backend config), but traffic between proxy and backend is unencrypted unless you re-encrypt (mTLS).
- **Complexity vs. transparency** — Adding a reverse proxy introduces another component to monitor, configure, and secure. For simple deployments, it may be unnecessary overhead.

### Real-world failure modes
- **Buffer overflow attacks** — If the proxy buffers large request bodies without limits, attackers can exhaust memory.
- **Misconfigured upstream timeouts** — Too-long timeouts tie up proxy connections; too-short ones cause premature failures for legitimate slow operations.
- **Open proxy misconfiguration** — If misconfigured, it can become a forward proxy, letting external users route arbitrary traffic through your infrastructure.

### When to choose
- Use when you need **SSL termination, static caching, or to hide backend topology**. NGINX or Envoy in front of app servers is the standard pattern. If you already have an API gateway, it likely serves this role too.

---

## 4. Rate Limiting

Controls how many requests a client can make in a given time window. Protects services from abuse, DDoS, and noisy neighbors in multi-tenant systems. Common algorithms:
- **Fixed Window** — simple but bursty at window edges
- **Sliding Window Log/Counter** — smoother but more memory
- **Token Bucket** — allows controlled bursts
- **Leaky Bucket** — enforces strict uniform rate

Typically implemented at the gateway layer with a distributed counter (Redis). Returns **HTTP 429** when exceeded.

### Trade-offs
- **Fixed Window vs. Sliding Window** — Fixed window is cheap (one counter per window) but allows **2x burst at window boundaries** (e.g., 100 requests at 11:59 + 100 at 12:00). Sliding window log is precise but stores every timestamp. Sliding window counter is the sweet spot.
- **Token Bucket vs. Leaky Bucket** — Token bucket allows **controlled bursts** (good for real user traffic patterns). Leaky bucket enforces **strict uniform rate** (good for protecting fragile backends). Choose based on whether burstiness is acceptable.
- **Local vs. Distributed** — Per-node limits are fast but inaccurate (10 nodes × 100/s = 1000/s global). Distributed (Redis + Lua script) is accurate but adds latency per request (~1ms). At very high scale, use local limits with periodic sync.

### Real-world failure modes
- **Redis goes down** — Do you fail open (allow all traffic, risking backend overload) or fail closed (reject all, causing outage)? Most systems fail open with local fallback limits.
- **Key design** — Rate limiting by IP fails for corporate NATs (thousands of users behind one IP). Limit by **authenticated user ID** when possible, IP as fallback.
- **Missing rate limits on internal services** — A misconfigured cron job or batch process can DDoS your own service.

### When to choose
- **Every public API needs rate limiting.** Internall services need it too, but can be more relaxed. Implement at the **gateway layer** for simplicity, but critical services should also have **local rate limits** as defense-in-depth.

---

## 5. Throttling

Often conflated with rate limiting, but throttling is the *enforcement mechanism* — **degrading service gracefully** rather than hard-rejecting. Strategies include slowing response times, serving from cache, reducing response fidelity, or queuing requests. It's about **graceful degradation** under load. Example: an API returns lower-resolution images when the system is under stress.

### Trade-offs
- **Hard rejection vs. graceful degradation** — Rate limiting says "no"; throttling says "yes, but less." Throttling keeps the user experience alive (even if degraded) vs. hard 429s that break client flows.
- **Complexity** — Implementing tiered degradation (full response → cached response → partial response → error) requires planning each degradation level.
- **Priority-based throttling** — Premium users get full service; free-tier users get degraded. This requires knowing the user tier at the edge, adding latency to lookup.

### Real-world failure modes
- **Degraded mode becoming the default** — If the system is chronically under-provisioned, users permanently get degraded responses. Monitoring must alert on throttle rate.
- **Inconsistent degradation** — If different services degrade differently without coordination, the user sees a Frankenstein response (high-res images + missing recommendations).

### When to choose
- Use when you need **availability over correctness** — e-commerce product pages, social media feeds. Don't use for financial transactions where partial results are dangerous.

---

## 6. Pagination

Technique to break large result sets into manageable chunks. Three main approaches:
- **Offset-based** (`LIMIT 20 OFFSET 40`) — simple but suffers from drift on inserts and poor perf on large offsets (DB still scans skipped rows)
- **Cursor-based / Keyset** (`WHERE id > last_seen_id LIMIT 20`) — performant and stable, ideal for infinite scroll
- **Page-token** — server encodes an opaque token (cursor + filters), giving full control over iteration semantics

For high-scale feeds (Twitter, Slack), cursor-based is the standard.

### Trade-offs
- **Offset vs. Cursor** — Offset is simple and supports "jump to page 50" but is O(n) in the DB (it scans and skips rows). Cursor is O(1) but only supports forward/backward navigation — no random page access.
- **Consistency** — Offset pagination drifts when rows are inserted/deleted between pages (duplicate or missing items). Cursor pagination is stable since it's anchored to a specific record.
- **Opaque token vs. exposed cursor** — Opaque tokens let you change implementation without breaking clients. Exposed cursors (e.g., `?after=id_123`) are simpler but lock you into that scheme.

### Real-world failure modes
- **Deep pagination attacks** — `?offset=10000000` forces the DB to scan millions of rows. Always set a max offset or use cursors.
- **Cursor invalidation** — If the record the cursor points to is deleted, cursor-based pagination can break. Use a "deleted" soft-delete or handle gracefully.

### When to choose
- **Cursor-based for feeds, timelines, infinite scroll** (anything high-volume, append-heavy). **Offset for admin dashboards** where users need "page 5 of 20" and data is relatively static.

---

## 7. Cache Stampede

When a popular cache key expires, **hundreds of concurrent requests simultaneously miss cache and hit the database**, potentially crushing it. Mitigations:
- **Locking / Singleflight** — only one request recomputes; others wait
- **Stale-while-revalidate** — serve stale data while one thread refreshes
- **Probabilistic early expiration (XFetch)** — randomly refresh before TTL
- **Pre-warming** — proactively populate before expiry

Critical in any system where cache hit ratio matters (e.g., product pages, feed timelines).

### Trade-offs
- **Locking (mutex)** — Simple and effective: one thread recomputes, others wait. But introduces **lock contention** and a brief latency spike for waiting requests.
- **Stale-while-revalidate** — No latency spike (serves stale data) but you serve **potentially outdated data** for a brief window. Acceptable for feeds, not for bank balances.
- **Probabilistic early expiration (XFetch)** — Elegant, no locks, statistically prevents stampede. But adds complexity and the probability tuning is non-trivial.

### Real-world failure modes
- **Celebrity problem** — A single viral post's cache expires, and millions of requests stampede. Even locking doesn't help if the recompute takes 5 seconds — the lock queue grows unbounded. Solution: **never expire hot keys; use background refresh.**
- **Cold start stampede** — After a cache flush or deployment, *everything* is a miss simultaneously. Solution: **cache warming** before routing traffic.

### When to choose
- **Any system with hot keys and high QPS.** If your cache is small and low-traffic, TTL is fine. At scale, you need at least one stampede prevention mechanism.

---

## 8. Idempotency

An operation is idempotent if **executing it multiple times produces the same result as executing it once**. Essential for safe retries in distributed systems. HTTP GET, PUT, DELETE are idempotent by design; POST is not. Implementation: client sends an **idempotency key** (UUID) with the request; server stores the result keyed by that ID and returns the stored result on duplicates. Critical for payment systems — you never want to charge a customer twice because of a network retry.

### Trade-offs
- **Storage cost** — You must store idempotency keys and their results for some retention window. For high-throughput APIs, this can be significant.
- **Retention window** — Too short, and late retries create duplicates. Too long, and storage grows. Typically 24–72 hours.
- **Scope** — Idempotency at the API layer doesn't guarantee end-to-end idempotency if downstream services aren't idempotent. You need it at **every layer that can be retried**.

### Real-world failure modes
- **Missing idempotency on payments** — Network timeout → client retries → double charge. This is the canonical example. Stripe, PayPal, etc. all require idempotency keys.
- **Non-deterministic idempotency keys** — If the client generates a new key on each retry (bug), idempotency is useless. The key must be generated **once per logical operation**.
- **Race condition** — Two requests with the same key arrive simultaneously. Without proper locking (e.g., `INSERT ... ON CONFLICT`), both may execute. Use DB-level uniqueness constraints.

### When to choose
- **Mandatory for any state-mutating API** in distributed systems — payments, order placement, inventory updates. Optional for read-only or naturally idempotent operations (PUT with same body).

---

## 9. GraphQL

A query language for APIs developed by Facebook. Clients **declare exactly what data they need** in a single request, solving REST's over-fetching and under-fetching problems. The server exposes a **typed schema**; clients compose queries against it. Trade-offs: complex caching (no URL-based caching), potential for expensive queries (need query depth/cost limiting), and the N+1 problem requires **DataLoader batching**. Best fit for client-heavy apps with diverse data needs (mobile, dashboards).

### Trade-offs
- **Flexibility vs. complexity** — Clients get exactly what they need, but you need query cost analysis, depth limiting, and complexity budgets to prevent expensive queries.
- **Caching** — REST has URL-based HTTP caching out of the box. GraphQL uses POST for queries, so you need **persisted queries or a normalized cache** (Apollo Client).
- **N+1 problem** — A nested query (`users → posts → comments`) can trigger thousands of DB calls. **DataLoader** (batching + caching per request) is mandatory.
- **Schema evolution** — Adding fields is easy (non-breaking). Removing them requires deprecation cycles. Versioning is implicit, unlike REST's explicit `/v1/` `/v2/`.

### Real-world failure modes
- **Denial of service via deep queries** — A malicious client requests `{ user { friends { friends { friends { ... } } } } }` 20 levels deep. Solution: query depth limit + cost analysis.
- **Over-reliance on a single endpoint** — Monitoring is harder. With REST, you see which endpoint is slow. With GraphQL, you need **per-resolver tracing** (Apollo Tracing).

### When to choose
- Best for **client-heavy apps with diverse frontends** (mobile needs less data than web). Overkill for simple CRUD APIs or internal service-to-service communication.

---

## 10. gRPC

A high-performance RPC framework by Google using **HTTP/2 and Protocol Buffers** (binary serialization). Supports **unary, server-streaming, client-streaming, and bidirectional-streaming** calls. ~10x more compact and faster than JSON/REST. Contract-first via `.proto` files enables strong typing and auto-generated client/server stubs in multiple languages. Ideal for **internal service-to-service communication** in microservices. Downsides: not natively browser-friendly (need gRPC-Web), harder to debug (binary), and tighter coupling to schema.

### Trade-offs
- **Performance vs. debuggability** — Protobuf is ~10x faster/smaller than JSON, but you can't `curl` a gRPC endpoint or inspect payloads in browser DevTools easily.
- **Streaming vs. complexity** — Bidirectional streaming is powerful for real-time use cases, but error handling in streaming is significantly more complex than request-response.
- **Tight coupling** — Both client and server must share `.proto` files. Schema changes require regenerating stubs. Less flexible than REST for public APIs.
- **HTTP/2 requirement** — Some legacy infrastructure (older load balancers, proxies) doesn't support HTTP/2 well.

### Real-world failure modes
- **Load balancer incompatibility** — HTTP/2 multiplexes on one connection, so L4 LBs send all traffic to one backend. You need **L7-aware gRPC LB** or client-side balancing.
- **Deadline propagation** — If Service A calls B calls C, and A's deadline isn't propagated, C may continue working after A has already timed out. Always propagate deadlines.

### When to choose
- **Internal service-to-service communication** where performance matters. **Not** for public-facing browser APIs (use REST or GraphQL). Use gRPC-Web as a bridge if needed.

---

## 11. Webhooks

An **inversion of the polling model** — instead of the consumer repeatedly asking "anything new?", the producer pushes an HTTP POST to a registered callback URL when an event occurs. Used heavily in integrations (Stripe payment events, GitHub push events). Design considerations: **retry with exponential backoff**, **signature verification (HMAC)**, **idempotent receivers**, and **dead-letter queues** for failed deliveries.

### Trade-offs
- **Push vs. pull** — Webhooks are real-time and efficient (no wasted polling requests). But the provider must handle **delivery reliability** — retries, DLQs, ordering.
- **Receiver availability** — If the receiver is down, events are lost unless the provider retries. Providers typically retry with exponential backoff for 24–72 hours.
- **Security** — Anyone can send a POST to your webhook URL. You **must** verify signatures (HMAC-SHA256 with a shared secret).

### Real-world failure modes
- **Thundering herd on receiver** — A provider sends thousands of webhook events at once (e.g., bulk import completes). The receiver must be able to handle bursts or the provider should support throttled delivery.
- **Ordering** — Webhooks may arrive out of order. If you get `order.shipped` before `order.created`, your system must handle it (store-and-forward, or re-fetch state).
- **Replay attacks** — Without timestamp validation, an attacker can replay captured webhook payloads.

### When to choose
- Use for **event-driven integrations** (payment notifications, CI/CD triggers). For internal event-driven architecture, prefer a **message broker** (Kafka, SQS) for durability and ordering guarantees.

---

## 12. JWT

A **self-contained, signed token** (Base64-encoded JSON) used for stateless authentication. Structure: `Header.Payload.Signature`. The server signs it with a secret (HMAC) or private key (RSA/ECDSA); consumers verify without calling the auth server. **Pros**: no session store, horizontal scaling is trivial. **Cons**: can't be revoked easily (must wait for expiry or maintain a blacklist, defeating statelessness). Best practice: short-lived access tokens + long-lived refresh tokens stored securely.

### Trade-offs
- **Stateless vs. revocability** — The biggest tension. JWTs are stateless (no DB lookup), but you **can't revoke them** without maintaining a blacklist — which reintroduces state.
- **Token size** — JWTs are large (hundreds of bytes with claims) compared to opaque session IDs. They're sent on every request, inflating bandwidth.
- **Short-lived access + refresh token pattern** — Mitigates revocation problem. Access tokens live 5–15 min, refresh tokens live days/weeks and are stored server-side.

### Real-world failure modes
- **Storing JWTs in localStorage** — Vulnerable to XSS. Store in **httpOnly, Secure, SameSite cookies** instead.
- **Weak signing algorithm** — `alg: none` attack: the server accepts unsigned tokens. Always whitelist allowed algorithms server-side.
- **Bloated tokens** — Putting too much data in claims (roles, permissions, user profile) makes the token huge and leaks info if intercepted.

### When to choose
- **Stateless API authentication** for horizontally scaled services. Not ideal when you need **instant revocation** (e.g., "user changed password, invalidate all sessions") — use opaque tokens + session store for that.

---

## 13. OAuth 2.0

An **authorization framework** (not authentication) that allows a third-party application to access a user's resources **without exposing credentials**. Key flows:
- **Authorization Code + PKCE** — standard for web/mobile apps
- **Client Credentials** — machine-to-machine
- **Device Code** — for input-constrained devices (smart TVs)

Issues: token leakage, redirect URI manipulation. Pair with **OpenID Connect (OIDC)** for authentication (ID tokens). Understanding the difference between OAuth (authorization) and OIDC (authentication) is a senior-level signal.

### Trade-offs
- **Security vs. complexity** — OAuth is a framework, not a protocol. The spec has many flows, extensions, and subtle security requirements. Misimplementation is common.
- **Authorization Code + PKCE vs. Implicit** — Implicit flow (tokens in URL fragment) is deprecated due to token leakage. PKCE is now mandatory even for public clients.
- **Token storage on mobile** — Refresh tokens on mobile must be stored in secure enclave / keychain, not SharedPreferences / UserDefaults.

### Real-world failure modes
- **Open redirect** — If the redirect URI isn't strictly validated, an attacker redirects the auth code to their server. Always use **exact redirect URI matching**.
- **Token leakage via referrer header** — If the redirect page loads external resources, the token in the URL may leak via `Referer`. Use `Referrer-Policy: no-referrer`.
- **Confused deputy** — Without OIDC's `aud` claim, a token from one service can be replayed to another. Always validate audience.

### When to choose
- **Whenever third-party access is needed** (social login, API integrations). For internal-only auth between services, mTLS or service-account JWTs are simpler.

---

## 14. Cache Invalidation

> *"There are only two hard things in CS: cache invalidation and naming things."*

Strategies:
- **TTL (Time-To-Live)** — simple, eventual consistency
- **Write-through** — write to cache + DB atomically; strong consistency but higher write latency
- **Write-behind (write-back)** — write to cache, async flush to DB; fast writes, risk of data loss
- **Cache-aside (lazy loading)** — app manages cache explicitly; most common pattern
- **Event-driven invalidation** — DB change events (CDC / Debezium) trigger cache purge; best for consistency at scale

The right strategy depends on your **consistency requirements vs. performance budget**.

### Trade-offs
- **Write-through vs. cache-aside** — Write-through gives strong consistency but doubles write latency. Cache-aside is simpler and more common, but there's a race condition: if a read happens between the DB write and cache delete, stale data gets cached.
- **Event-driven (CDC) invalidation** — The gold standard for consistency. DB changes trigger cache purge via Debezium/Kafka. But adds infrastructure complexity (CDC pipeline, Kafka, consumer).
- **TTL vs. explicit invalidation** — TTL is simple but data is stale until expiry. Explicit invalidation is precise but requires knowing *all* the cache keys affected by a data change (hard with complex queries).

### Real-world failure modes
- **Delete-then-write race condition** (cache-aside): Thread A writes to DB → Thread B reads from DB (old value, due to replication lag) → Thread B populates cache with stale data → Thread A deletes cache (too late, B already wrote stale). Solution: **delayed double-delete** or CDC-based invalidation.
- **Cache inconsistency across regions** — Multi-region caches can have different values. Solutions: regional TTL, cross-region invalidation bus.

### When to choose
- **Cache-aside + TTL** for most use cases. **CDC-based** for systems where consistency is critical (inventory, pricing). **Write-through** only when read-after-write consistency is mandatory and write volume is low.

---

## 15. Composite Index

A database index on **multiple columns** `(A, B, C)`. Follows the **leftmost prefix rule** — the index can serve queries on `(A)`, `(A, B)`, or `(A, B, C)`, but **not** `(B)` or `(C)` alone. Column order matters enormously: place **high-selectivity, equality-matched columns first**, range columns last. A well-designed composite index can turn a full table scan into an **index-only scan (covering index)**, eliminating disk reads entirely. Misunderstanding column order is one of the most common perf mistakes in production.

### Trade-offs
- **Read vs. write performance** — Every index speeds up reads but slows down writes (index must be updated on INSERT/UPDATE/DELETE). Too many composite indexes can kill write throughput.
- **Index size** — Composite indexes on wide columns (VARCHAR(255), VARCHAR(255), VARCHAR(255)) consume significant disk/memory. This can push the index out of RAM, defeating its purpose.
- **Covering index trade-off** — Including all SELECT columns in the index (`INCLUDE` clause in PostgreSQL) avoids table lookup entirely but bloats the index.

### Real-world failure modes
- **Wrong column order** — An index on `(status, created_at)` helps `WHERE status = 'active' ORDER BY created_at`. But `WHERE created_at > X AND status = 'active'` may not use it efficiently because `created_at` is a range condition on the second column.
- **Index not used** — The optimizer may skip your index if it estimates the query will return >20-30% of rows (full scan is cheaper). Functions on indexed columns (`WHERE LOWER(email) = ...`) also bypass the index — use functional indexes.

### When to choose
- When you have **queries that filter/sort on multiple columns together frequently**. Analyze slow query logs, not guesswork. Every index should justify its existence with a real query pattern.

---

## 16. Query Optimization

The process of making database queries efficient. Key levers:
- **EXPLAIN/ANALYZE** — read the execution plan; look for seq scans, high row estimates, sort spills
- **Indexing** — B-tree, hash, GIN, GiST depending on query pattern
- **Avoiding SELECT *** — reduce I/O
- **Denormalization** — trade write complexity for read speed
- **Partitioning** — prune irrelevant data
- **Query rewriting** — EXISTS vs. IN, JOINs vs. subqueries
- **Connection pooling** (PgBouncer) — reduce connection overhead

At senior level, you should be able to read an execution plan and pinpoint where time is spent.

### Trade-offs
- **Normalization vs. denormalization** — Normalized schemas save storage and avoid update anomalies, but complex JOINs are expensive. Denormalize read-heavy, rarely-updated data.
- **Index everything vs. selective indexing** — Over-indexing kills writes and wastes memory. Index based on actual slow query log analysis.
- **Materialized views** — Pre-computed query results. Fast reads, but stale until refreshed. Good for dashboards, bad for real-time data.

### Real-world failure modes
- **Implicit type casting** — `WHERE varchar_column = 123` forces a cast on every row, bypassing the index. Always match types.
- **ORM-generated queries** — ORMs (Hibernate, Django ORM) generate suboptimal queries (N+1, unnecessary JOINs, SELECT *). Always review generated SQL in production.
- **Statistics staleness** — Query planners rely on table statistics. If not updated (`ANALYZE`), the planner makes bad choices (e.g., nested loop instead of hash join).

### When to choose
- **Always ongoing.** Query optimization isn't a one-time task. Set up slow query logging, review weekly, and optimize the top offenders. The 80/20 rule applies: 20% of queries cause 80% of load.

---

## 17. CAP Theorem

In a **distributed data store**, you can only guarantee **two of three**: **Consistency, Availability, Partition Tolerance**. Since network partitions are inevitable, the real choice is **CP vs. AP**:
- **CP** (e.g., ZooKeeper, etcd, HBase) — rejects requests during partition to stay consistent
- **AP** (e.g., Cassandra, DynamoDB) — remains available but serves potentially stale data

In practice, modern systems use **tunable consistency** (e.g., Cassandra's quorum reads/writes) to slide along the spectrum. Also worth mentioning **PACELC** — even when no partition, there's a trade-off between **Latency and Consistency**.

### Trade-offs
- **CP systems** (ZooKeeper, etcd, Spanner) — You get consistency but sacrifice availability during partitions. Good for **leader election, distributed locks, configuration**.
- **AP systems** (Cassandra, DynamoDB) — Always available, eventually consistent. Good for **high-write workloads where stale reads are acceptable** (social media likes, IoT telemetry).
- **Tunable consistency** — Cassandra lets you choose per-query: `ONE` (AP), `QUORUM` (middle ground), `ALL` (CP). This is the practical reality — you don't choose once for the whole system.

### Real-world failure modes
- **Split brain** — In a CP system without proper consensus, a network partition can create two leaders, both accepting writes. On heal, data conflicts. This is why **Raft/Paxos** exist.
- **Misunderstanding "Available"** — CAP's "Available" means every non-failing node returns a response. It doesn't mean "the system as a whole is up." A CP system can still have high uptime — just not during partitions.

### When to choose
- **Don't choose one for the whole system.** Different data has different requirements. User sessions → AP. Financial ledger → CP. The interview-winning answer is: "It depends on the data, and I'd use different datastores for different consistency requirements."

---

## 18. ACID

Properties of reliable database transactions:
- **Atomicity** — all or nothing (via WAL / undo logs)
- **Consistency** — transaction moves DB from one valid state to another (constraints enforced)
- **Isolation** — concurrent transactions don't interfere (levels: Read Uncommitted → Serializable)
- **Durability** — committed data survives crashes (fsync, WAL)

In distributed systems, achieving ACID across nodes requires **2PC or consensus protocols (Raft/Paxos)**, which is expensive. That's why many distributed DBs relax to **BASE** (Basically Available, Soft-state, Eventually consistent).

### Trade-offs
- **Isolation levels vs. performance** — Serializable is safest but has the worst throughput (locks or MVCC overhead). Read Committed is the default in PostgreSQL and is sufficient for most workloads. Repeatable Read prevents phantom reads but increases lock contention.
- **ACID vs. BASE** — In distributed systems, full ACID requires 2PC (Two-Phase Commit) which is slow and fragile (coordinator failure blocks everyone). BASE (eventual consistency + compensating transactions via Sagas) is the practical alternative.
- **Single-node ACID vs. distributed ACID** — PostgreSQL gives you ACID for free. Across microservices, you must explicitly build it via **Saga pattern** (choreography or orchestration).

### Real-world failure modes
- **Long-running transactions** — Hold locks for minutes, blocking all other writes. Solution: break into smaller transactions, use optimistic locking.
- **Lost updates** — Two concurrent reads, both modify, both write — the second write overwrites the first. Solution: optimistic concurrency control (version column + `WHERE version = X`).
- **2PC coordinator failure** — Participants are stuck in "prepared" state, holding locks indefinitely. This is why 2PC is largely abandoned in favor of Sagas.

### When to choose
- **Single-database transactions: always ACID.** Cross-service: use **Sagas with compensating actions**. Reserve distributed ACID (e.g., Google Spanner's TrueTime) for cases where you truly need global consistency and can afford the infrastructure.

---

## 19. Sharding

Horizontal partitioning of data across multiple database nodes. Each shard holds a subset of the data. Strategies:
- **Range-based** — easy to implement, prone to hotspots
- **Hash-based** — even distribution, but range queries become scatter-gather
- **Directory-based** — lookup table maps key → shard; flexible but the directory is a bottleneck

Challenges: **cross-shard joins, rebalancing when adding nodes, distributed transactions**. Consistent hashing (with virtual nodes) minimizes data movement during rebalancing. Sharding is a last resort — exhaust vertical scaling, read replicas, and caching first.

### Trade-offs
- **Hash-based vs. range-based** — Hash gives even distribution but kills range queries (must scatter-gather across all shards). Range is great for time-series (`WHERE date BETWEEN ...`) but creates hotspots (the latest shard gets all writes).
- **Application-level vs. proxy-level** — App-level sharding (you write the routing logic) is flexible but complex. Proxy-level (Vitess, ProxySQL, Citus) abstracts it but adds latency and operational complexity.
- **Rebalancing** — Adding a shard requires moving data. With consistent hashing + virtual nodes, only ~1/n of data moves. Without it, you may need to rehash everything.

### Real-world failure modes
- **Hotspot shard** — One celebrity user or viral product generates disproportionate traffic to one shard. Solution: further split hot shards, or use a secondary sharding key.
- **Cross-shard queries** — `SELECT * FROM orders JOIN users` when orders and users are on different shards requires scatter-gather + app-level join. Extremely expensive. Co-locate related data on the same shard.
- **Distributed transactions** — An order affecting inventory (shard A) and payments (shard B) needs 2PC or Saga. Complexity explodes.

### When to choose
- **Last resort.** Try vertical scaling, read replicas, caching, and query optimization first. Shard when a single node can't handle write throughput or data volume. Instagram sharded PostgreSQL at ~1B rows per table.

---

## 20. Circuit Breaker

A **stability pattern** (from Michael Nygard's *Release It!*) that prevents cascading failures. States:
- **Closed** — requests flow normally; failures are counted
- **Open** — after threshold breached, all requests **fail fast** without calling the downstream service
- **Half-Open** — after a cooldown, a few probe requests are allowed; if they succeed, circuit closes

Implementations: Resilience4j, Hystrix (deprecated), Envoy's outlier detection. Pairs with **retries (with jitter), timeouts, and fallbacks**. Without circuit breakers, one slow downstream service can exhaust thread pools and take down the entire call chain.

### Trade-offs
- **Fail-fast vs. fallback** — Fail-fast (return error immediately) is simple. A fallback (cached response, default value, degraded feature) is better UX but harder to implement and test.
- **Threshold tuning** — Too sensitive → circuit opens on transient errors, causing unnecessary outages. Too lenient → slow failures cascade before the circuit trips. Use **failure rate + slow call rate** thresholds together.
- **Half-open probing** — How many probe requests to allow? Too few → slow recovery. Too many → if the downstream is still broken, probes add load.

### Real-world failure modes
- **Missing circuit breakers** — Service A calls B calls C. C is slow (not failing). Without a circuit breaker, A and B exhaust thread pools waiting for C. The entire chain dies. This is the #1 cause of cascading failures.
- **Monitoring gaps** — If circuit state isn't monitored/alerted, the circuit may stay open for hours without anyone noticing.
- **Retry storm** — Circuit breaker + aggressive retries = amplified load on recovery. Always pair with **exponential backoff + jitter**.

### When to choose
- **Every synchronous service-to-service call.** No exceptions. Use a library (Resilience4j) or service mesh (Istio's outlier detection). Async messaging (Kafka) doesn't need circuit breakers since the broker decouples producer and consumer.

---

## 21. Livelock

A condition where **two or more threads are actively responding to each other but making no progress** — unlike deadlock where threads are blocked, in livelock they keep *changing state* without advancing. Classic analogy: two people in a hallway who keep stepping aside in the same direction. In systems, it happens when retry logic conflicts — e.g., two transactions keep aborting and retrying, interfering with each other each time. Fix: **add randomized backoff / jitter** to break symmetry.

### Trade-offs
- **Backoff randomization** — Adding jitter breaks symmetry but adds unpredictable latency. A purely deterministic system might be more predictable but more prone to livelock.
- **Priority-based resolution** — Assign priorities to competing threads/transactions. The lower-priority one backs off. Simple but can cause starvation.
- **Detection vs. prevention** — Livelocks are harder to detect than deadlocks (threads are "busy," not blocked). Monitoring must track **throughput, not just liveness**.

### Real-world failure modes
- **Optimistic locking livelock** — Two transactions repeatedly read-modify-write the same row. Both detect a conflict, both retry immediately, both conflict again. Solution: random backoff.
- **Network protocol livelock** — Two Ethernet devices detect collision, both back off the same amount, both retransmit, collide again. Ethernet uses **binary exponential backoff** to prevent this.
- **Resource contention in message processing** — Two consumers keep requeueing messages that conflict with each other. Neither makes progress.

### When to choose
- Livelock isn't something you "choose." It's something you **design against** whenever you have retry logic, optimistic concurrency, or mutual resource negotiation. The key defense is always **randomized backoff**.

---

## 22. CSRF

An attack where a malicious site tricks a user's browser into making an **authenticated request** to a trusted site (since the browser auto-attaches cookies). Example: an `<img>` tag whose `src` is a bank transfer URL. Mitigations:
- **Synchronizer Token Pattern** — server-generated token embedded in forms, validated on submission
- **SameSite cookie attribute** — `Strict` or `Lax`
- **Double-submit cookie** — token in both cookie and header; attacker can't read the cookie cross-origin
- **Custom request headers** — browsers don't allow cross-origin custom headers in simple requests

### Trade-offs
- **Synchronizer token vs. SameSite cookies** — Synchronizer tokens work everywhere but require server-side storage/validation and don't work for pure APIs. `SameSite=Strict` is zero-effort but breaks legitimate cross-site navigation (e.g., clicking a link from email).
- **SameSite=Lax** — Allows top-level GET navigations (links from email work) but blocks cross-site POST/PUT. Good default, but doesn't protect GET-based state changes (which shouldn't exist, but do in legacy apps).
- **Double-submit cookie** — Stateless (no server-side token store), but requires the client JS to read the cookie and set a header, which won't work if JS is blocked.

### Real-world failure modes
- **GET endpoints with side effects** — `GET /transfer?amount=1000&to=attacker` is CSRF-able even with POST-only token protection. **Never mutate state on GET.**
- **Subdomain cookie leakage** — If your cookie is set on `.example.com`, a compromised subdomain (`evil.example.com`) can exploit it. Scope cookies tightly.
- **Token fixation** — If the CSRF token doesn't rotate on login, an attacker who knows a pre-auth token can use it post-auth.

### When to choose
- **Every web application with cookie-based authentication** needs CSRF protection. APIs that use Bearer tokens in the `Authorization` header are not vulnerable (browser doesn't auto-attach headers). If you're building a SPA with token-based auth, CSRF is less of a concern; XSS becomes the primary threat.

---

## 23. Backpressure

A **flow-control mechanism** where a slow consumer signals the producer to slow down, preventing unbounded queue growth and OOM. Without backpressure, fast producers overwhelm slow consumers. Implementations:
- **Reactive Streams** (Project Reactor, RxJava) — subscriber requests N items
- **TCP flow control** — receiver advertises window size
- **Message brokers** — consumer ack-based delivery (RabbitMQ), consumer pull (Kafka)
- **Application-level** — return HTTP 429/503, use bounded queues that reject when full

Critical in streaming architectures and async pipelines. The goal is **system stability over throughput**.

### Trade-offs
- **Drop vs. buffer vs. slow down** — Three strategies when the consumer is overwhelmed. Dropping is the fastest but loses data. Buffering delays processing but risks OOM. Slowing down the producer preserves data but reduces throughput.
- **Pull-based vs. push-based** — Kafka's pull model naturally applies backpressure (consumer controls pace). Push-based systems (RabbitMQ, WebSockets) need explicit flow control.
- **Bounded vs. unbounded queues** — Unbounded queues hide backpressure until OOM. **Always use bounded queues** and decide what to do when full (reject, block, drop oldest).

### Real-world failure modes
- **OOM kills** — The #1 symptom of missing backpressure. A producer-consumer pipeline with unbounded queues eventually eats all memory when the consumer slows down.
- **Cascading slowdown** — Without backpressure, a slow downstream service causes upstream services to queue up, consuming memory and threads, until the whole system freezes.
- **Load shedding gone wrong** — Aggressive backpressure (rejecting too early) wastes available capacity. Tuning the pressure threshold is critical.

### When to choose
- **Any asynchronous pipeline or streaming system.** Kafka handles it inherently (consumer pull). For reactive systems, use Reactive Streams (Project Reactor, RxJava). For HTTP, return 429/503 and let the client retry with backoff.

---

## 24. False Sharing

A **CPU cache performance problem**. Modern CPUs cache data in **cache lines (typically 64 bytes)**. When two threads on different cores modify **different variables that happen to share the same cache line**, the cache coherence protocol (MESI) forces the line to bounce between cores — called **cache line ping-pong**. Neither thread is writing the other's data, but performance degrades to worse-than-single-threaded. Fix: **pad data structures** so hot variables land on separate cache lines (`@Contended` in Java, `alignas` in C++). Relevant in high-throughput concurrent data structures (e.g., Disruptor, concurrent counters).

### Trade-offs
- **Padding vs. memory efficiency** — Padding variables to separate cache lines wastes memory (64 bytes per padded variable). In most applications this doesn't matter, but in memory-constrained embedded systems, it might.
- **Language support** — Java has `@Contended` (JDK 8+, requires `-XX:-RestrictContended`). C++ has `alignas(64)`. Go doesn't have a direct annotation — you manually pad structs.
- **Detection difficulty** — False sharing doesn't cause incorrect behavior, just poor performance. It's invisible to debuggers. You need **hardware performance counters** (perf, VTune, perf stat) to detect cache line bouncing.

### Real-world failure modes
- **Concurrent counters** — An array of per-thread counters where adjacent elements share a cache line. The "independent" counters contend. Solution: `LongAdder` in Java, or pad each counter.
- **LMAX Disruptor** — Their high-performance ring buffer explicitly pads sequence counters to avoid false sharing. This was a key insight in achieving millions of ops/sec.
- **Struct layout in Go/C** — Two frequently-written fields adjacent in a struct shared by goroutines/threads. Reorder fields or add padding.

### When to choose
- Only matters in **ultra-high-throughput, low-latency concurrent systems** — financial exchanges, high-perf queues, game engines. For 99% of web services, false sharing is irrelevant. But knowing it exists signals deep systems understanding in an interview.

---

## 25. mTLS

Standard TLS only authenticates the **server** to the client. mTLS adds **client certificate verification** — both sides present and validate certificates. Used for **zero-trust service-to-service communication** in microservice meshes (Istio, Linkerd). Each service gets a short-lived certificate (via SPIFFE/SPIRE or a service mesh CA). Benefits: **identity verification, encryption in transit, no shared secrets**. Operationally complex — requires certificate lifecycle management (issuance, rotation, revocation). It's the backbone of modern zero-trust networking.

### Trade-offs
- **Security vs. operational complexity** — mTLS provides the strongest service identity guarantee, but you need a **PKI (Public Key Infrastructure)**: a CA, certificate issuance, rotation, and revocation. Manual cert management doesn't scale — use automated solutions (cert-manager, SPIFFE/SPIRE).
- **Performance** — TLS handshake adds latency (~1-2ms). With connection reuse and session resumption, this is amortized. But in microservices with many short-lived connections, it can be noticeable.
- **Service mesh abstraction** — Istio/Linkerd handle mTLS transparently at the sidecar level. Apps don't need to know about certificates. Trade-off: sidecar adds ~1ms latency per hop and consumes resources.

### Real-world failure modes
- **Certificate expiry** — The #1 operational failure. If cert rotation fails silently, services can't communicate after expiry. **Always monitor cert expiry with alerts.**
- **CA compromise** — If the root CA's private key is leaked, all certificates are compromised. Use intermediate CAs with short-lived certs to limit blast radius.
- **Clock skew** — Certificates have `notBefore` and `notAfter` timestamps. If a server's clock is skewed, valid certs appear invalid. Use NTP everywhere.

### When to choose
- **Zero-trust environments, multi-tenant platforms, and any system handling sensitive data.** If you're running a service mesh (Istio, Linkerd), mTLS comes nearly free. Without a mesh, implement it for critical service-to-service paths (e.g., auth service ↔ user service) and expand gradually.

---

## Interview Strategy

For **any** of these concepts in a senior system design interview, follow this formula:

1. **Define it clearly** in 1-2 sentences
2. **Explain the trade-offs** — there's no silver bullet
3. **Give a real-world failure mode** you've seen or studied
4. **State when you'd choose it** and when you wouldn't

This demonstrates not just knowledge but **engineering judgment**, which is what senior-level interviews are evaluating.

---

## Quick Reference Checklist

- [ ] API Gateway: Centralization vs SPOF, latency, BFF pattern
- [ ] Load Balancer: L4 vs L7, algorithms, client-side vs server-side
- [ ] Reverse Proxy: SSL termination, caching, security
- [ ] Rate Limiting: Fixed vs sliding window, token vs leaky bucket, local vs distributed
- [ ] Throttling: Graceful degradation, priority tiers
- [ ] Pagination: Offset vs cursor, consistency, deep pagination attacks
- [ ] Cache Stampede: Locking, stale-while-revalidate, celebrity problem
- [ ] Idempotency: Storage cost, retention window, race conditions
- [ ] GraphQL: Flexibility vs complexity, N+1, caching challenges
- [ ] gRPC: Performance vs debuggability, streaming, HTTP/2 requirements
- [ ] Webhooks: Push vs pull, delivery reliability, security
- [ ] JWT: Stateless vs revocability, token size, storage
- [ ] OAuth 2.0: Security vs complexity, PKCE, token storage
- [ ] Cache Invalidation: Write-through vs cache-aside, CDC, TTL
- [ ] Composite Index: Read vs write performance, column order, covering indexes
- [ ] Query Optimization: Normalization vs denormalization, ORM pitfalls
- [ ] CAP Theorem: CP vs AP, tunable consistency
- [ ] ACID: Isolation levels, distributed ACID vs Sagas
- [ ] Sharding: Hash vs range, rebalancing, cross-shard queries
- [ ] Circuit Breaker: Fail-fast vs fallback, threshold tuning
- [ ] Livelock: Randomized backoff, detection challenges
- [ ] CSRF: Synchronizer token vs SameSite, GET mutations
- [ ] Backpressure: Drop vs buffer vs slow down, bounded queues
- [ ] False Sharing: Padding, detection, when it matters
- [ ] mTLS: Security vs complexity, service mesh, cert management

---

**Good luck with your interviews!** 🚀
