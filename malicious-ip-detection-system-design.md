# Malicious IP Detection System - System Design

## Problem Statement

Design a system that can detect and block malicious IP addresses for an enterprise, including geo-distributed blacklists, edge processing capabilities, and efficient filtering mechanisms to prevent attacks at scale.

A Malicious IP Detection System is a network security service that identifies and blocks bad IP addresses before they reach your applications. Think of it like the IP reputation and blocking features in a WAF/CDN or API gateway, but purpose-built for enterprise scale with geo-distributed blacklists, edge enforcement, and probabilistic filters for ultra-low-latency checks.

---

## 1. Functional Requirements

1. **IP Validation**: System must check the source IP address of every incoming request against a dedicated Malicious IP Service
2. **Near Real-Time Updates**: Handle updates to the Malicious IP Service with propagation delay ≤ 5 minutes (okay up to 5 mins for the update of Malicious IPs to reflect)
3. **Access Control**: Malicious IPs should be denied of services
4. **IP Management**: Support adding and removing IPs from the blacklist with proper authorization (client - end users should be checked if they have access to add new IPs)

---

## 2. Non-Functional Requirements

1. **Scale**: Handle 100k read requests per second (~10B reads per day)
2. **Read-Heavy Workload**: Write:Read ratio = 1:100 (read >> write; 100k read rps → 10B reads per day)
3. **Low Latency**: p95 latency < 50ms to detect if an IP is blacklisted or not
4. **Availability > Consistency**: Eventual consistency acceptable; it's okay to store some stale data up to 5 minutes
5. **High Availability**: Geo-distributed deployment for fault tolerance and low latency
6. **Security**: Authenticated writes to prevent unauthorized IP additions/removals

---

## 3. Core Entities

### IP Address
- `ipAddress` (string): IPv4/IPv6 address
- `addedAt` (timestamp): When IP was blacklisted
- `addedBy` (string): User/service that added it
- `metadata` (optional): Reason, severity, expiry

### Client/User
- `userId` (string): Identifier
- `role` (enum): Permissions for IP management operations
- `jwt` (string): Authentication token for authorization

---

## 4. API Routes

### Read API (High Volume - 100k rps)
```http
GET /maliciousIp?ipaddress=xxx
Response: { "isBlacklisted": true } or { "isBlacklisted": false }
Status: 200 OK
```

### Write API (Low Volume - Authenticated)
```http
POST /maliciousIP
Body: {
  "jwt": "...",
  "ipAddress": "xxx.xxx.xxx.xxx"
}
Response: 201 Created
```

### Delete API (Low Volume - Authenticated)
```http
DELETE /maliciousIP?ipaddress=xxx
Headers: { "Authorization": "Bearer <jwt>" }
Response: 200 OK
```

---

## 5. High-Level Design

```
┌─────────┐                    ┌──────────────────────┐
│ Client  │───── 403/200 ─────→│  API Gateway         │
└─────────┘                    │  Load Balancer       │
                               │  (Geo-replicated)    │
                               └──────────┬───────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                    ▼                     ▼                     ▼
           ┌─────────────────┐   ┌──────────────┐    ┌──────────────────┐
           │  Rate Limiter   │   │Auth Service  │    │  Org Wide        │
           │     (429)       │   │   (JWT)      │    │  Services        │
           └─────────────────┘   └──────────────┘    └──────────────────┘
                                          │
                                          ▼
                                   ┌─────────────┐
                                   │    Kafka    │
                                   │  (Events)   │
                                   └──────┬──────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │  Malicious IP Service │
                              │  (Radix Trie lookup)  │
                              └───────────┬───────────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │   Redis Cluster       │
                              │   (Geo-replicated)    │
                              │   - Async replication │
                              │   - TTL + LRU         │
                              │   - 2-3 nodes/region  │
                              └───────────────────────┘
```

### Architecture Components

#### 1. **Client Layer**
- End users or services making requests
- Receives 403 (blocked) or 200 (allowed) responses

#### 2. **API Gateway / Load Balancer**
- **Geo-replicated** for low latency across regions
- **Rate limiting** to prevent abuse (returns 429 if rate exceeded)
- **Fast path**: Queries Redis for IP lookup
- Routes 403 if IP is blacklisted, 200 if clean

#### 3. **Authentication Service** (Write/Delete only)
- Validates JWT tokens
- Verifies user permissions for IP management
- Only for POST/DELETE operations, not for reads

#### 4. **Message Queue (Kafka)**
- Handles write/delete operations **asynchronously**
- Provides event replay for auditing
- Decouples control plane from data plane
- Retention policy for compliance and debugging
- **Only for Write/Delete IP operations** (not for reads)

#### 5. **Malicious IP Service** (Control Plane)
- Master source of truth for IP blacklist
- Processes writes from Kafka
- Publishes updates to Redis clusters globally
- Uses **Radix Trie** for efficient IP prefix matching
- Handles batch processing

#### 6. **Redis Cluster** (Data Plane - Read Service)
- **Purpose**: Fast IP lookups for read requests
- **Geo-replicated** across regions (2-3 nodes per region: 1 primary + 1-2 followers)
- **Async replication** for eventual consistency
- **TTL + LRU eviction strategy** to manage memory
- Handles hot IP reads with random char appended to IP as cache key
- **Batch read operations** to reduce latency
- **Sharding**: Only if dataset gets big (PB scale) for memory management and scaling out

---

## Request Flows

### Read Path (Fast - < 50ms)
1. Client request → Geo-distributed Load Balancer
2. Load Balancer queries local Redis cluster (synchronous)
3. Redis returns blacklist status from cache
4. Load Balancer returns **403** (blocked) or **200** (allowed)

**Key Point**: We don't want the Load Balancer to wait for a response if the Malicious IP Service is down, so we replicate to Redis globally and async.

### Write Path (Async - eventual consistency)
1. Admin/automated system sends POST request → Load Balancer
2. **Auth Service** validates JWT and permissions
3. Write event published to **Kafka** (asynchronous)
4. **Malicious IP Service** consumes event from Kafka
5. Service updates master database
6. Change propagates to **Redis clusters globally** (≤ 5 min)
7. Returns **201 Created** immediately (doesn't wait for propagation)

### Delete Path (Async - similar to write)
1. Admin sends DELETE request → Load Balancer
2. Auth Service validates JWT and permissions
3. Delete event published to Kafka
4. Malicious IP Service processes deletion
5. Removal propagates to Redis clusters
6. Returns **200 OK**

---

## 6. Deep Dive

### 6.1 Data Structure Choice: Radix Trie vs Bloom Filter

#### Radix Trie (✅ Selected)
- ✅ **Supports deletions** (critical requirement)
- ✅ Efficient IP prefix matching (CIDR ranges like 192.168.0.0/24)
- ✅ **No false positives** (accurate results)
- ✅ Space-efficient for IP addresses: O(n × prefix_length)
- ✅ Fast lookups: O(key_length)

**Use Case**: Malicious IP Service uses Radix Trie for lookup operations.

#### Bloom Filter (❌ Rejected)
- ✅ Ultra-low memory footprint
- ✅ Very fast lookups (O(k) where k = hash functions)
- ❌ **Cannot handle deletions** (dealbreaker for our use case)
- ❌ False positives possible (can incorrectly flag clean IPs as malicious)
- ❌ Requires full rebuild to remove items

**Conclusion**: We chose Radix Trie because **bloom filters cannot deal with deletion of IPs**, which is a core requirement.

---

### 6.2 Caching Strategy (Redis Cluster)

#### Configuration
- **Replication**: Async cross-region replication for low latency
- **Topology**: 2-3 nodes per region (1 primary, 1-2 replicas/followers)
- **Eviction Policy**: TTL (e.g., 24 hours) + LRU for memory management
- **Hot Reads**: Handle using random char appended to IP address as cache key to distribute load
- **Batching**: Batch read APIs (check multiple IPs in one request) to reduce network round trips

#### Why Geo-Replicate?
- **Low latency**: Local Redis cluster in each region provides < 50ms reads
- **High availability**: Region failures don't affect other regions
- **Async replication**: Acceptable trade-off for 5-minute propagation delay

#### When to Shard?
- **Shard only if dataset gets big (PB scale)**
- Current scale: millions of IPs (~100-500MB) fits easily in memory
- Shard only for **memory management and scaling out**, not prematurely

---

### 6.3 Geo-Distribution & Consistency

#### Trade-off: Availability > Consistency
- **Async replication** means different regions may have slightly different blacklists
- **Up to 5 min lag** acceptable per requirements
- Better to block a malicious IP 5 minutes late than have high latency or unavailability
- **Eventual consistency** model

#### Propagation Flow
1. Write/Delete event published to Kafka
2. Kafka events consumed by regional Malicious IP Services
3. Each region updates its local Redis cluster independently
4. Global consistency achieved within **≤ 5 minutes**

---

### 6.4 Asynchronous Request Handling

#### Why Kafka for Writes?
1. **Decoupling**: Control plane (write operations) separate from data plane (read operations)
2. **Auditing**: Kafka retention allows replaying/auditing events
3. **Scalability**: Can handle more traffic in future if write volume increases
4. **Resilience**: If downstream service is slow, Kafka buffers events
5. **Ordering**: Maintains event order within partitions

#### Benefits
- Write/Delete operations don't block
- System can handle traffic spikes
- Failed operations can be retried automatically
- Complete audit trail for compliance

---

### 6.5 Failure Modes & Mitigations

#### Redis Cluster Down
**Problem**: Local Redis unavailable
**Solutions**:
1. Fall back to Malicious IP Service directly (slower, but available)
2. Return cached result from CDN/API Gateway memory
3. **Fail-open** (allow traffic) vs **fail-closed** (block traffic) - depends on risk tolerance

#### Kafka Lag/Failure
**Problem**: Events not processing fast enough
**Mitigations**:
- Monitor consumer lag; alert if > 5 minutes
- Auto-scaling consumers based on lag
- Replay capability ensures no events lost
- Dead letter queue for failed events

#### Hot IP Attack
**Problem**: Attacker uses single IP for millions of requests
**Mitigations**:
- Redis handles reads efficiently (100k+ rps per node)
- **Rate limiter** blocks excessive requests from single IP (429 response)
- Use **random character appended to IP** as cache key to distribute load across cache nodes
- Circuit breaker to prevent cascade failures

#### Network Partition
**Problem**: Regions can't communicate
**Mitigations**:
- Each region operates independently
- Async replication resumes when network recovers
- Local Redis ensures region remains functional

---

### 6.6 Observability & Safe Rollout

#### Key Metrics
- **Cache hit rate**: Redis hit/miss ratio
- **p95/p99 latency**: For IP lookups
- **Propagation delay**: Time from write → visible in all regions
- **False positive/negative rates**: Monitor blocking accuracy
- **Kafka consumer lag**: Ensure < 5 minute SLA
- **Redis memory usage**: Prevent evictions of active IPs

#### Logging & Monitoring
- Log all blocking decisions (IP, timestamp, reason)
- Distributed tracing for request flows
- Alerting on anomalies (sudden spike in blocks, high error rates)

#### Safe Rollout Strategy
1. **Shadow mode**: Log blocks without enforcing (validate accuracy)
2. **Gradual rollout**: Enable for 1% → 10% → 50% → 100% of traffic
3. **Kill switch**: Ability to disable blocking globally in emergencies
4. **A/B testing**: Compare block rates between old and new systems
5. **Canary deployments**: Deploy to one region first

---

### 6.7 Capacity Estimation

#### Traffic
- **Reads**: 100k rps = 8.64B reads/day (~10B/day)
- **Writes**: 1k rps = 86.4M writes/day (~100M/day)

#### Storage
- **IP count**: ~10M malicious IPs
- **Storage per IP**: ~50 bytes (IP + metadata)
- **Total storage**: 10M × 50 bytes = **500MB**
- **Conclusion**: Fits in single Redis instance, no immediate sharding needed

#### Bandwidth
- **Read bandwidth**: 100k rps × 1KB response = **100 MB/s** = 0.8 Gbps
- **Write bandwidth**: 1k rps × 1KB = **1 MB/s** = 8 Mbps

#### Redis Cluster Sizing
- **Nodes per region**: 2-3 (1 primary + 1-2 replicas)
- **Regions**: ~10 global regions
- **Total Redis nodes**: 20-30 globally
- **Memory per node**: 1-2 GB (plenty of headroom)

---

### 6.8 Alternative Considerations

#### 1. CDN-Based Filtering
- **Approach**: Push blacklist to edge (CloudFlare, Fastly)
- **Pros**: Sub-millisecond checks, DDoS protection built-in
- **Cons**: Vendor lock-in, less control, propagation delays

#### 2. Hardware Acceleration
- **Approach**: Use eBPF/XDP at kernel level
- **Pros**: Ultra-low latency (microseconds), no userspace overhead
- **Cons**: Complex to implement, Linux-specific

#### 3. ML-Based Detection
- **Approach**: Train models to predict malicious IPs proactively
- **Pros**: Can identify new threats before they're blacklisted
- **Cons**: False positives, model training complexity, drift

#### 4. CIDR Range Support
- **Approach**: Store IP ranges (e.g., 192.168.0.0/24) instead of individual IPs
- **Pros**: Massive storage reduction for block ranges
- **Cons**: More complex lookup logic, potential over-blocking

#### 5. Bloom Filter as L1 Cache
- **Approach**: Use Bloom filter for initial fast check, Radix Trie for confirmation
- **Pros**: Ultra-fast negative lookups (clean IPs)
- **Cons**: Additional complexity, false positive handling

---

## Key Interview Talking Points

### 1. **Control Plane vs Data Plane Separation**
- **Control plane** (Kafka + Malicious IP Service): Handles writes, updates, management
- **Data plane** (Redis + Load Balancer): Handles high-volume reads with low latency
- This separation allows independent scaling and reliability

### 2. **Consistency Trade-offs**
- Chose **eventual consistency** over strong consistency
- Acceptable 5-minute propagation delay balances latency and freshness
- AP system (Availability + Partition tolerance) over CP system

### 3. **Adversarial Considerations**
- **Rate limiting** prevents single IP from overwhelming system
- **Authentication** prevents attackers from manipulating blacklist
- **Audit logs** (Kafka) provide forensics
- **Hot IP handling** prevents cache stampede

### 4. **Scalability Path**
- Current design handles 100k rps comfortably
- Can scale to 1M+ rps by:
  - Adding more Redis replicas
  - Sharding Redis by IP range
  - Adding more API Gateway instances
  - Increasing Kafka partitions

### 5. **Why Not Just Use a Database?**
- Traditional databases too slow for 100k rps reads (even with indexes)
- Redis provides in-memory speed with persistence
- Radix Trie optimized for IP prefix matching
- Database good for control plane, not data plane

---

## Summary

This Malicious IP Detection System design achieves:
- ✅ **100k rps** read throughput with **< 50ms p95 latency**
- ✅ **Geo-distributed** architecture for global low latency
- ✅ **5-minute propagation** for blacklist updates
- ✅ **Availability over consistency** (eventual consistency)
- ✅ **Efficient data structures** (Radix Trie for deletions)
- ✅ **Asynchronous writes** (Kafka for decoupling)
- ✅ **Horizontal scalability** (Redis sharding when needed)

The system separates fast-path reads (Redis cache) from slow-path writes (Kafka + master service), uses geo-replication for low latency, and employs Radix Trie instead of Bloom filters to support IP deletions while maintaining accuracy.
