# Google Object Storage System Design (Google Cloud Storage / Colossus)

## 1. What is object storage? (start here)

An **object store** is the simplest possible storage abstraction:

```
PUT  bucket/key  →  store this blob (+ metadata)
GET  bucket/key  →  give me the blob back
DELETE, LIST     →  that's basically the whole API
```

- **Objects are immutable** — you never edit byte 5 of an object; you upload a new version. No POSIX, no directories (the "/" in keys is cosmetic), no random writes.
- Why so restrictive? Immutability removes the hardest distributed-systems problems (concurrent in-place writes, locking, cache invalidation) and unlocks *practically infinite* scale. This is why data lakes, ML datasets, backups, and static content all live in object stores.
- **Google Cloud Storage (GCS)** is Google's offering (Amazon's is S3). GCS promises **11 nines durability** (99.999999999%) and **strong consistency**: after a successful write, *every* subsequent read/list sees it — notable because S3 was eventually-consistent until 2020.

---

## 2. The layered design (the big picture to draw in an interview)

GCS is a thin API over a stack Google built across 20 years:

```
┌────────────────────────────────────────────────────┐
│  GCS API front-ends (HTTP/JSON, auth, quotas)      │   ← stateless, load-balanced
├────────────────────────────────────────────────────┤
│  Metadata layer: bucket/object metadata in         │
│  Spanner (Paxos-replicated SQL database)           │   ← strong consistency comes from here
├────────────────────────────────────────────────────┤
│  Data layer: object bytes in Colossus              │
│  (successor of GFS — cluster-level file system)    │
├────────────────────────────────────────────────────┤
│  D file servers + physical disks/SSD               │
└────────────────────────────────────────────────────┘
```

Design principle #1: **separate metadata from data**. Metadata (small, needs transactions) goes to Spanner; bytes (huge, need bandwidth) go to Colossus. A `GET` does a metadata lookup, then streams bytes directly from storage servers — the control plane never touches the data path.

---

## 3. Colossus — the heart of the design

Colossus is Google's successor to GFS (Google File System) and is *the* universal storage layer under GCS, Bigtable, Spanner, YouTube, Gmail, everything.

### Why GFS had to be replaced
GFS had **one master** holding all metadata in RAM → capped at ~billions of files and forced 64 MB chunks. Google needed trillions of objects, including tiny ones.

### Colossus's key moves

1. **Distributed, scalable metadata**: metadata is sharded across many **curators**, and — the beautiful recursive trick — stored in **Bigtable**, which itself stores its data on Colossus. (The recursion bottoms out: the lowest level's metadata is small enough for a Chubby/Paxos-managed core.)
2. **Client-driven replication**: the client library writes/encodes data directly to storage servers ("D servers"); curators only coordinate placement. Control/data separation, again.
3. **Erasure coding by default** (e.g., Reed–Solomon): durability at ~1.3–1.5× overhead instead of 3× replication → enormous cost savings at Google scale. Hot data may additionally be replicated for read bandwidth.
4. **Background maintenance**: rebalancing, repair of lost fragments, and migration run constantly; disks are assumed to be dying at all times.
5. **Storage tiering within a cluster**: newly written / hot data lands on flash; **custodians** demote cold data to spinning disks. Applications just see "a file."

---

## 4. Life of a request (walk through this in interviews)

### PUT gs://my-bucket/photo.jpg
1. Client hits a **global front-end** (nearest Google edge, via anycast); authenticated & authorized (IAM).
2. Front-end streams the bytes into **Colossus** in the bucket's home region — data is chunked, erasure-coded, fragments placed on many D servers across failure domains (different racks/power/network).
3. Object **metadata row committed to Spanner** (name → generation number, checksums, fragment locations, ACLs). This commit is the *linearization point* — it's what makes GCS strongly consistent.
4. Ack to client. Every later read/list is served consistent with this commit.

### GET
1. Front-end → metadata lookup in Spanner → fragment locations.
2. Stream fragments from D servers (reconstruct from any k of k+m if some are down), verify **checksums** (end-to-end integrity — corrupt fragments detected and repaired), return bytes.

### Object versioning / no in-place update
An overwrite creates a new **generation**; metadata swaps atomically in Spanner. Readers of the old generation keep reading it (immutability makes this trivially safe). Deletes are metadata operations; space is reclaimed by background garbage collection.

---

## 5. Multi-region, availability, and durability

- **Location types**: regional (one region), dual-region, multi-region buckets. Dual/multi-region → data is **asynchronously replicated across regions** (with a "recovery point objective" of minutes), plus metadata replicated via Spanner.
- Within a region, data spans **≥3 zones** — a whole-building failure loses nothing (see [../03-reliability-failover/datacenter-failover.md](../03-reliability-failover/datacenter-failover.md)).
- **Durability (11 nines)** comes from: erasure coding across failure domains + continuous checksum scrubbing + fast automatic repair. Availability (99.9–99.95% SLA) is a *different, lower* number — the system may briefly refuse requests without ever losing bytes. Interview gold: *durability ≠ availability*.
- **Storage classes** = price/latency tiers on the same API: Standard, Nearline, Coldline, Archive; lifecycle policies auto-demote aging objects.

---

## 6. Scalability tricks worth naming

- **Stateless front-ends** → scale horizontally behind Google's global load balancer.
- **Hotspot management**: object keys are range-sharded in the metadata layer; sequential key names (e.g., timestamps) can hotspot a shard — GCS auto-splits ranges, but key-name randomization is still a documented best practice.
- **Resumable & parallel composite uploads** for big objects; **compose** operation stitches 32 objects into one.
- **Caching/CDN integration** for hot public objects.

---

## 7. Interview questions & strong answers

**Q: Design S3/GCS from scratch — where do you start?**
A: Split metadata (transactional DB, Paxos-replicated) from data (chunked, erasure-coded blob store). Stateless API tier in front. Strong consistency = make the metadata commit the single linearization point. Then discuss durability math, repair, and multi-region replication.

**Q: Why is object storage cheaper and more scalable than a filesystem?**
A: Immutability + flat namespace remove locking, hierarchical directory contention, and in-place-update complexity; erasure coding cuts cost; every component shards horizontally.

**Q: How does GCS give strong consistency when S3 historically didn't?**
A: All metadata mutations commit through Spanner (globally consistent, TrueTime-ordered). Reads consult the same authority. S3 originally used an eventually-consistent metadata layer and retrofitted strong consistency in 2020.

**Q: What happens when a disk dies?**
A: Nothing visible. Reads reconstruct from remaining erasure-code fragments; background repair re-creates lost fragments on other disks within minutes. Durability is active repair, not disk trust ([distributed-storage.md](distributed-storage.md)).

---

## 8. Key takeaways

1. Object store = **immutable blobs + flat namespace** → the trade that buys infinite scale.
2. GCS = **API front-ends + Spanner metadata + Colossus data** — separation of concerns at every layer.
3. **Colossus** fixed GFS by sharding metadata (stored in Bigtable, recursively on Colossus) and erasure-coding data.
4. Strong consistency lives in **one place**: the metadata commit.
5. Say it in interviews: *durability ≠ availability*; both are engineered, with different mechanisms.
