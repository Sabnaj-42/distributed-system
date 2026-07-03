# Distributed Storage

## 1. What and why

**Distributed storage** = storing data across many machines while presenting it as one logical system.

A single server hits three walls:
1. **Capacity** — disks are finite (a petabyte doesn't fit in one box).
2. **Throughput** — one machine's network card and disks can only move so many bytes/sec.
3. **Fault tolerance** — one machine = one power supply away from losing everything.

Distributed storage solves all three with two core ideas: **partitioning** (split the data) and **replication** (copy the data).

---

## 2. The flavors of storage (know the taxonomy)

| Type | Abstraction | Examples | Typical use |
|---|---|---|---|
| **Block storage** | Raw virtual disk (read/write block N) | AWS EBS, Ceph RBD, Google Persistent Disk | VM disks, databases |
| **File storage** | Hierarchy of files/directories, POSIX-ish | HDFS, GFS/Colossus, NFS, CephFS, Lustre | Analytics, shared filesystems, HPC |
| **Object storage** | Flat key → blob (+ metadata), HTTP API | Amazon S3, **Google Cloud Storage**, Ceph RGW, MinIO | Backups, images/video, data lakes, ML datasets |
| **Distributed databases/KV** | Records, queries, transactions | Spanner, Cassandra, DynamoDB, CockroachDB | Application data |

Object storage trades away in-place edits and POSIX semantics (objects are immutable; you replace the whole object) for **massive scalability and simplicity** — that's why the cloud runs on it. Details: [google-object-storage.md](google-object-storage.md).

---

## 3. Core idea #1 — Partitioning (sharding)

Split data so each node holds a piece.

- **Range partitioning**: keys A–F → node 1, G–M → node 2… Great for range scans; risk of **hotspots** (everyone writing keys with today's date hits one node). Used by Bigtable, HBase, Spanner.
- **Hash partitioning**: `node = hash(key) mod N`. Even spread, but naïve `mod N` reshuffles *everything* when N changes. Fix: **consistent hashing** — nodes and keys map onto a ring; adding a node moves only ~1/N of the data. Used by Dynamo, Cassandra. Ceph's **CRUSH** algorithm goes further: it computes placement from the cluster map, so there's *no lookup table at all*.
- A central **metadata service** (GFS master, HDFS NameNode) is the alternative: a directory that remembers where every chunk lives. Simpler, strongly consistent, but must itself be made highly available.

---

## 4. Core idea #2 — Replication

Keep k copies (typically 3) so failures don't lose data.

- **Leader-based (primary/backup)**: writes go to a leader, which orders them and pushes to followers. Simple, consistent; leader is a bottleneck/single point until failover.
- **Quorum-based (leaderless)**: write to W of N replicas, read from R; if R+W > N, reads see the latest write. (Dynamo/Cassandra style.)
- **Consensus-based**: each shard is a Raft/Paxos group; writes commit via majority. Strongest guarantees (Spanner, CockroachDB). See [../01-fundamentals/raft-and-paxos.md](../01-fundamentals/raft-and-paxos.md).

**Replication geometry:** replicas are placed across **failure domains** — different disks, machines, racks, and (optionally) data centers/zones — so one power failure can't take out all copies.

### Erasure coding — replication's cheaper sibling
3× replication costs 200% overhead. **Erasure coding** (like RAID, generalized) splits an object into *k* data fragments + *m* parity fragments; any *k* of the *k+m* reconstruct the object.
- Example: Reed–Solomon (6,3) → survives 3 losses at only 50% overhead (vs 3 losses needing 4× replication).
- Cost: reconstruction burns CPU and network; reads of degraded data are slower.
- Practice: hot/small data → replication; warm/cold/large data → erasure coding. **Google Colossus, S3, Ceph, HDFS-EC all do this.**

---

## 5. Consistency, failure detection, and repair

- **Consistency model**: what readers see mid-flight — strong/linearizable (acts like one copy) vs eventual (replicas converge later). Full discussion: [../01-fundamentals/cap-theorem.md](../01-fundamentals/cap-theorem.md).
- **Failure detection**: heartbeats + timeouts. Can't distinguish "dead" from "slow network" — a fundamental ambiguity handled with leases and quorums.
- **Anti-entropy / repair**: background jobs compare replicas (often via **Merkle trees** — hash trees that pinpoint differing ranges cheaply) and re-replicate under-replicated chunks. When a node dies, the system re-creates its replicas from the surviving copies — data durability is an *active* process, not a property of disks.
- **Durability math**: this is why S3/GCS advertise "11 nines" (99.999999999%) durability — the probability that all replicas/fragments die before repair completes is astronomically small.

---

## 6. Case study: GFS/HDFS architecture in 10 lines

The classic design every interview touches (Google File System, 2003 → HDFS is its open-source clone):

```
                 ┌──────────────┐
   metadata ops  │   Master /   │  (namespace, chunk → server map,
  ┌─────────────►│   NameNode   │   leases, rebalancing decisions)
  │              └──────┬───────┘
┌─┴────┐                │ heartbeats, chunk reports
│Client│         ┌──────┴──────────────────────┐
└─┬────┘         │  Chunkservers / DataNodes   │
  │   data I/O   │  (store 64MB chunks, 3×     │
  └─────────────►│   replicated across racks)  │
                 └─────────────────────────────┘
```

Key decisions and why:
- **Huge chunks (64 MB)** → tiny metadata, fits in master's RAM, few master interactions.
- **Data path bypasses the master** — clients talk to chunkservers directly; the master only hands out locations. The control/data plane split is *the* scalability trick.
- **Design for failure**: with thousands of commodity machines, failure is the normal case; replication + re-replication is constant background work.
- Weakness: one master limits total file count → Google replaced GFS with **Colossus** (sharded metadata, stored in Bigtable) — see [google-object-storage.md](google-object-storage.md).

---

## 7. Interview questions & strong answers

**Q: How do you store a 1 PB dataset reliably on machines with 4 TB disks?**
A: Partition into chunks, spread via consistent hashing or a metadata service, replicate 3× (or erasure-code) across racks, run background repair. Walk through GFS as the template.

**Q: Replication vs erasure coding — when each?**
A: Replication: simple, fast reads/repairs, 3× cost → hot data. EC: ~1.5× cost, CPU/network-heavy repairs → cold/large data. Real systems tier between them.

**Q: What breaks first as you scale?**
A: Usually metadata (the master/NameNode) — hence sharded or distributed metadata (Colossus, Ceph's CRUSH which eliminates the lookup entirely).

**Q: Why do all these systems separate metadata from data?**
A: Metadata is small but consistency-critical (needs consensus); data is huge but dumb (needs bandwidth). Different problems, different machinery.

---

## 8. Key takeaways

1. Two primitives explain everything: **partition** for scale, **replicate** for durability.
2. **Control plane vs data plane separation** is the master scalability trick.
3. Durability comes from **active repair**, not from disks being reliable.
4. **Erasure coding** = durability at ~1.5× cost instead of 3×.
5. Every design choice re-surfaces the CAP/latency trade-offs from [../01-fundamentals/cap-theorem.md](../01-fundamentals/cap-theorem.md).
