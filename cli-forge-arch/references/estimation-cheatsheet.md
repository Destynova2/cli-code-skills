# Back-of-Envelope Estimation Cheatsheet

Quick reference for capacity planning and system sizing. Use these formulas BEFORE designing architecture — the numbers drive the design.

---

## Latency Numbers Every Engineer Should Know

| Operation | Latency |
|-----------|---------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| RAM reference | 100 ns |
| SSD random read | 150 μs |
| HDD seek | 10 ms |
| Send 1 KB over 1 Gbps network | 10 μs |
| Read 1 MB sequentially from SSD | 1 ms |
| Read 1 MB sequentially from HDD | 20 ms |
| Read 1 MB sequentially from network | 10 ms |
| Round trip within same datacenter | 0.5 ms |
| Round trip cross-region (EU↔US) | 70-150 ms |
| Disk seek | 10 ms |
| TCP handshake | 1-3 ms (same DC) |
| TLS handshake | 5-50 ms |
| DNS lookup (uncached) | 20-120 ms |
| Redis GET | 0.1-0.5 ms |
| PostgreSQL simple query | 1-5 ms |
| PostgreSQL complex query | 10-100 ms |

**Key insight:** Memory is 1000x faster than SSD, SSD is 100x faster than HDD, same-DC network is 100x faster than cross-region.

---

## Unit Conversions

| From | To |
|------|----|
| 1 day | 86,400 seconds (~10^5) |
| 1 month | 2.6M seconds (~2.5 × 10^6) |
| 1 year | 31.5M seconds (~3 × 10^7) |
| 1 KB | 1,000 bytes (use 10^3) |
| 1 MB | 10^6 bytes |
| 1 GB | 10^9 bytes |
| 1 TB | 10^12 bytes |
| 1 PB | 10^15 bytes |
| 1 million | 10^6 |
| 1 billion | 10^9 |

**Tip:** Round aggressively. 86,400 ≈ 100,000. 2,592,000 ≈ 2.5M. Precision doesn't matter, order of magnitude does.

---

## Core Formulas

### 1. Traffic Estimation

```
Average QPS = DAU × requests_per_user_per_day ÷ 86,400

Peak QPS = Average QPS × peak_multiplier
  (peak_multiplier: 2x for uniform traffic, 3x for social apps, 5-10x for flash sales/events)

Read QPS = Total QPS × read_ratio ÷ (read_ratio + write_ratio)
Write QPS = Total QPS × write_ratio ÷ (read_ratio + write_ratio)
```

**Common read:write ratios:**
| System type | Read:Write |
|------------|-----------|
| Social media feed | 100:1 |
| Messaging | 10:1 |
| E-commerce | 50:1 |
| URL shortener | 100:1 |
| File storage | 5:1 |
| IoT ingestion | 1:10 |

### 2. Storage Estimation

```
Storage per day = new_objects_per_day × avg_object_size

Storage for N years = storage_per_day × 365 × N × replication_factor

Total with metadata = storage × (1 + metadata_overhead)
  (metadata_overhead: typically 10-20%)
```

**Common object sizes:**
| Data type | Size |
|-----------|------|
| Tweet/short text | 200-500 bytes |
| JSON API response | 1-10 KB |
| User profile | 1-5 KB |
| Thumbnail image | 20-50 KB |
| Photo (compressed) | 200 KB - 2 MB |
| Short video (1 min, compressed) | 5-50 MB |
| Log entry | 200-500 bytes |
| Database row (typical) | 500 bytes - 2 KB |

### 3. Bandwidth Estimation

```
Ingress bandwidth = Write QPS × avg_request_size
Egress bandwidth = Read QPS × avg_response_size

Total bandwidth = (Ingress + Egress) × overhead_factor
  (overhead_factor: 1.2 for headers, retries, etc.)
```

Conversion: 1 Gbps = 125 MB/s

### 4. Cache Estimation

```
Cache size = hot_data_percentage × total_data_size
  (hot_data_percentage: typically 20-30%, based on Pareto principle — 80/20 rule)

Alternative (request-based):
Cache size = peak_read_QPS × avg_response_size × cache_TTL_seconds
```

**Cache hit rate targets:**
| Tier | Hit rate | Typical use |
|------|---------|------------|
| L1 (in-process) | 95-99% | Very hot keys, config |
| L2 (Redis/Memcached) | 80-95% | User sessions, profiles |
| CDN | 90-99% | Static assets |

### 5. Server/Instance Estimation

```
Servers needed = Peak QPS ÷ QPS_per_server

With headroom:
Total servers = Servers needed × 1.3 (30% for failures, maintenance, deployment)

With availability:
Total servers = Servers needed × (1 ÷ (1 - failure_rate))
```

**Typical QPS per server:**
| Workload | QPS/server |
|----------|-----------|
| CPU-bound API (Go, Rust, Java) | 5,000 - 50,000 |
| CPU-bound API (Python, Ruby) | 500 - 5,000 |
| IO-bound API with DB queries | 1,000 - 10,000 |
| ML inference (GPU) | 10 - 1,000 |
| Static file serving | 10,000 - 100,000 |

### 6. Database Sizing

```
DB connections needed = (read_QPS × avg_query_time) + (write_QPS × avg_write_time)
  Example: (5000 × 0.005s) + (500 × 0.01s) = 25 + 5 = 30 connections

DB connections pool size = connections_needed × 1.5 (headroom)

Shards needed = total_data_size ÷ max_data_per_shard
  (PostgreSQL comfort zone: < 500GB per instance for good performance)
  (MongoDB: < 2TB per shard recommended)
```

---

## Availability Math

```
Availability = Uptime ÷ (Uptime + Downtime)

Error budget = 1 - SLO
```

| SLO | Downtime/year | Downtime/month | Downtime/day |
|-----|--------------|----------------|-------------|
| 99% (two 9s) | 3.65 days | 7.3 hours | 14.4 min |
| 99.9% (three 9s) | 8.76 hours | 43.8 min | 1.44 min |
| 99.99% (four 9s) | 52.6 min | 4.38 min | 8.6 sec |
| 99.999% (five 9s) | 5.26 min | 26.3 sec | 0.86 sec |

**Combined availability (serial):**
```
System = A × B × C
Example: API (99.9%) × DB (99.99%) × Cache (99.9%) = 99.8%
```

**Combined availability (with redundancy):**
```
Component = 1 - (1 - A)^N  (N = number of replicas)
Example: 2 replicas at 99.9% each = 1 - (0.001)^2 = 99.9999%
```

---

## Scalability Laws

### Little's Law
```
L = λ × W
  L = average number of items in system (connections, requests in flight)
  λ = arrival rate (requests/second)
  W = average time in system (latency)

Example: 1000 RPS × 0.1s avg latency = 100 concurrent requests
→ Need at least 100 connection pool size
```

### Amdahl's Law
```
Speedup(N) = 1 ÷ (S + (1-S)/N)
  S = fraction of work that is sequential (0 to 1)
  N = number of parallel processors

Example: S = 0.1 (10% sequential)
  Speedup(10) = 1 / (0.1 + 0.9/10) = 1 / 0.19 = 5.26x
  Speedup(100) = 1 / (0.1 + 0.9/100) = 1 / 0.109 = 9.17x
  Speedup(∞) = 1 / 0.1 = 10x max
→ 10% sequential work = max 10x speedup no matter how many nodes
```

### Universal Scalability Law (Gunther)
```
C(N) = N ÷ (1 + α(N-1) + βN(N-1))
  α = contention coefficient (serialization)
  β = coherence coefficient (crosstalk/coordination)
  N = number of processors/nodes

When β > 0, throughput DECREASES after a peak. This models real distributed systems.
```

---

## Quick Estimation Workflow

1. **Clarify scope:** users, geography, retention period
2. **Estimate traffic:** DAU → QPS → Peak QPS → Read/Write split
3. **Estimate storage:** objects/day × size × retention × replication
4. **Estimate bandwidth:** QPS × response sizes
5. **Estimate cache:** 20% of hot data, or request-based sizing
6. **Estimate servers:** Peak QPS ÷ QPS/server × 1.3
7. **Sanity check:** does the order of magnitude make sense?
8. **Document assumptions:** every number should have a stated assumption

---

## Common System Scales (Reference Points)

| System | DAU | QPS | Storage |
|--------|-----|-----|---------|
| Small SaaS (startup) | 10K | 10-100 | < 100 GB |
| Medium SaaS | 100K-1M | 100-10K | 100 GB - 10 TB |
| Large platform | 10M-100M | 10K-1M | 10 TB - 1 PB |
| Hyperscale (FAANG) | 1B+ | 1M+ | PB+ |
| Defense/sovereign platform | 1K-100K | 10-10K | 1 TB - 100 TB |
| IoT platform | N/A | 10K-1M (ingestion heavy) | 10 TB+ |

Use these as sanity checks. If your startup estimates need FAANG-level infrastructure, recheck your assumptions.
