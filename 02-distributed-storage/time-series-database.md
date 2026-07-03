# Time Series Databases (TSDB)

## 1. What is time series data?

A **time series** = a sequence of (timestamp, value) points for one measured thing.

```
cpu_usage{host="server-1", region="us-east"}:
  (12:00:00, 43.2), (12:00:15, 44.1), (12:00:30, 51.7), ...
```

The identity of a series = **metric name + label/tag set**. Examples everywhere:
- Infrastructure monitoring (CPU, memory, request latency) — Prometheus, Datadog
- IoT sensors (temperature, vibration)
- Finance (stock ticks), user analytics, weather

A **TSDB** is a database specialized for this shape: InfluxDB, Prometheus, TimescaleDB, VictoriaMetrics, Gorilla/Monarch (Facebook/Google internal), Amazon Timestream.

---

## 2. Why not just use PostgreSQL/MySQL?

Because time series workloads are *weird* in a very exploitable way:

| Property | Consequence |
|---|---|
| Writes are **append-only**, in rough time order | No updates/deletes in place → no B-tree random-write pain; LSM/columnar layouts shine |
| **Enormous ingest rate** (millions of points/sec) | Need write-optimized structures, batching |
| Recent data is hot; old data is rarely touched | Tiering: RAM → SSD → object storage |
| Queries = **ranges + aggregations** ("avg CPU per minute, last 6h") | Columnar scans, pre-aggregation beat point lookups |
| Old data loses value | **Retention policies** & **downsampling** are first-class features |
| Values change slowly between samples | **Delta compression** achieves 10–20× (~1.4 bytes/point in Gorilla) |

A general-purpose row-store B-tree database fights all of these; a TSDB is built around them.

---

## 3. Core techniques (the substance)

### 3.1 Storage layout: LSM trees + time partitioning
Most TSDBs use an **LSM (Log-Structured Merge) tree** pattern: writes go to an in-memory buffer (memtable) + WAL (write-ahead log for crash safety), then flush as immutable sorted files, later compacted. Sequential-only disk I/O → huge ingest.

Data is partitioned into **time blocks** (e.g., Prometheus: 2-hour blocks):
- Query "last 6h" touches only 3 blocks.
- **Retention = drop whole blocks** — deleting a billion points is one file unlink.

### 3.2 Compression — the signature trick (Facebook's Gorilla paper, 2015)
- **Timestamps**: samples arrive at ~regular intervals → store **delta-of-delta** (usually 0) in a few bits.
- **Values**: consecutive floats are similar → XOR consecutive values; the XOR is mostly zero bits; encode compactly.
- Result: **~1.37 bytes per 16-byte point** (~12×). This one paper is the ancestor of Prometheus's, InfluxDB's, and VictoriaMetrics's encodings.

### 3.3 Indexing: the inverted index over labels
To answer `cpu_usage{region="us-east", host=~"web-.*"}`, TSDBs keep an **inverted index**: label=value → list of series IDs (like a search engine). Intersect the posting lists, then fetch those series' blocks.

**Cardinality — the #1 operational problem**: every unique label combination is a new series. Adding a `user_id` label with 10M users × 5 metrics = 50M series → index and memory explode. Rule: labels must be *bounded* dimensions (host, region), never unbounded IDs. Expect this question if you mention monitoring.

### 3.4 Downsampling & retention
Keep raw 15s data for 2 weeks; store 5-min averages for 6 months; 1-hour rollups for 5 years. Old data answers "what was the trend," not "what was the exact value" — so pre-aggregate and delete raw. Continuous queries/recording rules automate this.

---

## 4. The distributed dimension (tie-in to this folder)

Scaling a TSDB past one node hits classic distributed-storage decisions ([distributed-storage.md](distributed-storage.md)):

- **Sharding**: usually **by series** (hash of metric+labels) so one series' data stays together; within a shard, partition by time. Sharding by time alone would hotspot "now."
- **Replication**: e.g., replication factor 2–3 across nodes/zones; monitoring systems often choose **AP over CP** (see [../01-fundamentals/cap-theorem.md](../01-fundamentals/cap-theorem.md)) — during a partition you'd rather ingest possibly-duplicated metrics than drop them; duplicates dedupe at query time. (Gorilla explicitly made this choice.)
- **Separation of compute & storage** (the modern architecture): Thanos, Cortex, Mimir, InfluxDB 3.0 keep recent data on ingest nodes and ship immutable blocks to **object storage** (S3/GCS — see [google-object-storage.md](google-object-storage.md)); stateless queriers fan out over both. Cheap infinite retention, independent scaling of ingest vs query.
- **Prometheus's contrarian design**: deliberately single-node, pull-based, no clustering — reliability through simplicity; scale by federation or by bolting on Thanos/Mimir.

```
writes → [ingesters (memtable+WAL)] → immutable blocks → [object storage]
queries → [queriers] ────────────────┴──────────────────────────┘
```

---

## 5. Interview questions & strong answers

**Q: Design a metrics system for 1M servers.**
A: Agents push (or are scraped) → sharded ingesters by series-hash, replication ×3 → Gorilla compression in memory (~26h of data fits in RAM per Gorilla's math) → flush 2h blocks to object storage → queriers + inverted label index → downsampling tiers + retention. Call out cardinality control and AP-during-partition ingestion.

**Q: Why can a TSDB compress 10–20× when general databases can't?**
A: It exploits domain structure: near-constant timestamp intervals (delta-of-delta ≈ 0) and slowly-changing values (XOR ≈ 0). General databases can't assume adjacent rows are correlated.

**Q: What's series cardinality and why does it kill TSDBs?**
A: Number of unique label-sets. Index size, memory, and churn scale with it, not with data volume. Unbounded labels (user IDs, request IDs) belong in logs/traces, not metrics.

**Q: Why is retention cheap in a TSDB and expensive in MySQL?**
A: Time-partitioned immutable blocks → expiry is dropping files. In a B-tree, deleting old rows is millions of random writes + vacuum.

---

## 6. Key takeaways

1. TSDB = database that **exploits append-only, time-ordered, range-queried** workload structure.
2. Four pillars: **LSM ingest, time-block partitioning, Gorilla compression, inverted label index**.
3. **Cardinality** is the failure mode everyone hits.
4. Modern architecture: **stateless ingest/query over object storage** — the same control/data separation as every system in this folder.
5. Monitoring TSDBs consciously choose **availability over consistency** — a perfect concrete CAP example to cite.
