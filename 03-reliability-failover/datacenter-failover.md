# Data Center Failover

## 1. The problem

Your entire application runs in one data center (DC). Then: a fiber cut, a power failure, a flood, a bad config push (most common!), or a cloud region outage. **Failover** = detecting that a site is unhealthy and shifting traffic and data to another site so users barely notice.

Two goals, two acronyms you must know:

| Term                                      | Meaning                            | Question it answers                       |
| ----------------------------------------- | ---------------------------------- | ----------------------------------------- |
| **RTO** — Recovery Time Objective  | Max acceptable downtime            | "How long until we're back up?"           |
| **RPO** — Recovery Point Objective | Max acceptable data loss (as time) | "How many minutes of writes can we lose?" |

Everything in DR (disaster recovery) is engineering RTO and RPO down — and every improvement costs money. RPO=0 and RTO≈0 is the most expensive corner.

---

## 2. The topology ladder (cheapest → most resilient)

1. **Backup & restore**: nightly backups shipped offsite. RTO hours–days, RPO up to 24h. Cheap.
2. **Pilot light**: minimal core (e.g., replicating database) always on in DC-B; spin up the rest on disaster.
3. **Warm standby**: full scaled-down copy running in DC-B; scale up + flip traffic. RTO minutes.
4. **Active–passive (hot standby)**: full-size replica receiving continuous data replication; only DC-A serves. RTO seconds–minutes.
5. **Active–active**: both DCs serve traffic simultaneously; failover = removing one from rotation. RTO ≈ 0, but now you own multi-site data consistency — the hardest version.

Cloud vocabulary: a **region** contains multiple **availability zones (AZs)** — physically separate buildings (power/network/cooling) close enough for ~1–2 ms latency. Modern default: **active–active across 3 AZs** (cheap, synchronous replication is feasible at 1 ms) + **cross-region DR** for the rare full-region event. Google/AWS design docs both push this two-tier pattern.

---

## 3. The data problem (this is where the distributed systems live)

Failing over stateless web servers is trivial. **State** is the hard part.

### Synchronous vs asynchronous replication

- **Synchronous**: commit only after DC-B confirms. **RPO = 0**, but every write pays inter-DC round-trip latency, and if the link degrades, you must choose: stall writes or drop to async — this is exactly the **CAP choice** ([../01-fundamentals/cap-theorem.md](../01-fundamentals/cap-theorem.md)). Feasible between AZs (~1 ms); painful across continents (~80+ ms).
- **Asynchronous**: commit locally, ship changes after. Fast writes, but on failover the replica lags → **RPO > 0** (you lose the unshipped tail) and you may serve stale reads.
- **Quorum/consensus across ≥3 sites**: Paxos/Raft groups spanning 3+ AZs/regions (Spanner, CockroachDB): any single site failure loses nothing and needs no "failover" at all — the quorum just keeps going. RPO=0 with availability, at the cost of majority-latency on every commit. See [../01-fundamentals/raft-and-paxos.md](../01-fundamentals/raft-and-paxos.md).

### Split-brain — the failure mode to fear

If DC-A is merely *unreachable* (not dead) and DC-B promotes itself, **both sides accept writes** → divergent data, possibly unmergeable. Defenses:

- **Quorum with an odd number of sites** (2 DCs + a lightweight tie-breaker "witness" in a third location is the classic pattern).
- **Fencing**: revoke the old primary's authority (kill its credentials, block it at the network, "STONITH") before promoting the new one.
- **Leases**: the primary's right to serve expires unless renewed; a partitioned primary stops accepting writes when its lease lapses — same mechanism as [../04-kubernetes-internals/kubernetes-lease.md](../04-kubernetes-internals/kubernetes-lease.md).

---

## 4. Storage-layer failover: the protocols & mechanisms

Section 3 framed *what* we want (RPO/RTO, avoid split-brain). This section is *how the storage layer actually does it* — the concrete replication protocols and the low-level mechanisms that make failover safe. Every real system is a combination of one **replication protocol** + a set of **coordination mechanisms**.

### 4.1 Replication protocols (how a write reaches the other copies)

| Protocol | How a write commits | Failover behavior | Real systems / paper |
| --- | --- | --- | --- |
| **Primary–backup (leader/follower)** | Primary applies, ships the change (log records) to backups; sync = wait for backup ack, async = don't | Detect dead primary → **promote a backup** → fence old one. Async ⇒ lose the unshipped log tail (RPO>0) | MySQL/Postgres replication, GFS |
| **Chain replication** | Write enters the **head**, propagates node→node down the chain, commits at the **tail** (which also serves reads) | Head dies → successor becomes head; tail dies → predecessor becomes tail; middle dies → chain splices out. Strong consistency, simple recovery | van Renesse & Schneider, OSDI 2004 |
| **CRAQ** (chain replication + apportioned queries) | Same write path as chain; every node keeps a *clean* + *dirty* version so **any node can serve consistent reads** | Same as chain replication, but read capacity scales with chain length | Terrace & Freedman, USENIX ATC 2009 |
| **Quorum / Dynamo-style** | Write to N replicas, ack after **W**; read from **R**; pick W+R>N for overlap | **No promotion step** — a down replica is just one missing vote; sloppy quorum + **hinted handoff** keep writes flowing; **Merkle-tree anti-entropy** repairs later | Dynamo, SOSP 2007; Cassandra |
| **Consensus (Raft / Paxos / Multi-Paxos)** | Leader appends to a replicated log, commits once a **majority** persists it | Leader loss → **election** picks a replica whose log is at least as up-to-date → **zero data loss**, no operator failover | Raft (ATC 2014); Spanner (OSDI 2012) — Paxos group per shard across ≥3 zones |
| **Log-shipping to a shared/disaggregated store** | DB tier ships only the **redo log** to a storage fleet; storage nodes materialize pages. Aurora: 6 copies over 3 AZs, **write quorum 4/6, read quorum 3/6** | Storage self-heals a lost node from quorum with no DB involvement; a failed DB node just re-attaches to the durable storage volume | Amazon Aurora, SIGMOD 2017 |

The trend the papers show: push durability *down* into the storage layer (quorum/consensus over the log) so that "failover" stops being a scary promote-and-pray event and becomes a routine majority operation.

### 4.2 The mechanisms that make it safe

These are the reusable building blocks the protocols above are assembled from:

- **Write-ahead / redo log shipping** — replicate the *log*, not the pages. "The log is the database" (Aurora): smaller network payload, and the log is the single source of truth for recovery and for catching up a lagging replica.
- **Quorum reads/writes (W + R > N)** — overlapping majorities guarantee a read sees the latest committed write without contacting every replica; tolerates up to N−W write / N−R read failures.
- **Leader election** — Raft/Paxos elect a new leader on timeout; the "up-to-date log" restriction is what makes promotion lossless (unlike naive async primary-backup).
- **Leases & epoch/term numbers** — a leader holds a time-bounded lease; every message carries a monotonically increasing epoch/term so a stale ex-leader's writes are rejected. This is the concrete anti-split-brain enforcement (see §3).
- **Fencing tokens (STONITH)** — the storage nodes refuse writes stamped with an old epoch, so a partitioned old primary that *thinks* it's still leader can't corrupt the volume.
- **Anti-entropy / read-repair (Merkle trees)** — background reconciliation of divergent replicas after a partition heals; how eventually-consistent stores converge without a coordinator.
- **Hinted handoff / sloppy quorum** — a temporarily-down replica's writes are parked on a stand-in node and replayed when it returns, so a node failure doesn't reduce write availability.
- **Gossip membership + failure detection** — decentralized "who is alive" so the cluster agrees on the replica set to route quorums to (Dynamo/Cassandra).

### 4.3 Worked example — how a storage failover actually sequences

Aurora on losing an AZ (from the SIGMOD 2017 paper): a write only needed **4 of 6** acks, so losing 2 replicas (a whole AZ + one more) still leaves a write quorum — **no failover at all**, just degraded redundancy. Losing the DB compute node triggers re-pointing to the durable 6-copy volume (RTO ≈ seconds), not a data copy. Contrast async primary-backup, where the same event means: detect → promote → fence → accept the RPO>0 tail loss.

That difference — *"failover as a data migration"* vs *"failover as a quorum/election that loses nothing"* — is the whole point of the storage-layer design choices, and the papers below are the primary sources.

---

## 5. Traffic failover — how requests move to DC-B

- **DNS-based** (Route 53, Cloud DNS + health checks): change the name → new IPs. Simple, global; but DNS caching/TTLs mean minutes of stragglers.
- **Anycast / global load balancers** (Google GLB, Cloudflare): one IP announced from many sites via BGP; failover is near-instant routing change, no client cooperation needed.
- **Health checking** drives all of it: probes must measure *user-visible* health (can I complete a real request?) not just "is the port open." Flapping protection: fail fast, recover slowly (hysteresis).

Failover sequencing matters: **fence old primary → promote data replica → verify → shift traffic**. Shifting traffic before the data layer is ready just moves the outage.

---

## 6. Hard-won operational lessons (interviewers love these)

1. **Untested failover doesn't exist.** DR that's never exercised fails when needed. Hence **chaos engineering**: Netflix Chaos Monkey (kill instances), Chaos Kong (evacuate an entire AWS region regularly, in production); Google's DiRT exercises.
2. **Capacity**: surviving DCs must absorb the failed DC's load. Run every site ≤ N-1/N capacity (e.g., 2 DCs → each ≤50% utilized), or failover triggers a cascading overload — the second outage is self-inflicted.
3. **The failover system itself must be simpler and more reliable than what it protects.** Control planes causing outages is a recurring postmortem theme.
4. **Correlated failures**: the disaster and its response can share a dependency (auth service, DNS, config store in the failed region). Facebook's 2021 BGP outage locked engineers out of the tools needed to fix it.
5. **Failback** (returning to DC-A) is a second migration and needs the same rigor; data written in DC-B must reconcile back.
6. **Practice regularly**: Google SRE's rule of thumb — if you haven't failed over recently, assume you can't.

---

## 7. Case study to cite: Google Spanner's approach

Instead of primary/standby failover, Spanner makes every data shard a **Paxos group across ≥3 zones/regions**. A zone loss is absorbed by the quorum with zero data loss and ~no downtime — "failover" is replaced by *continuous consensus*. Trade-off: cross-site commit latency on every write (the PACELC latency cost). This is the intellectual endpoint of the topology ladder.

---

## 8. Interview questions & strong answers

**Q: Design DR for a payment system.**
A: Payments ⇒ RPO=0 ⇒ synchronous or quorum replication across ≥3 AZs; cross-region async copy with an explicit, small RPO for region loss (or 3-region quorum for RPO=0 at latency cost); fencing + witness against split-brain; regular game-day drills; N+1 capacity.

**Q: Sync vs async replication?**
A: RPO=0 vs write latency — a direct CAP/PACELC instance. Sync within a region (1 ms), async or quorum across regions, chosen per data class.

**Q: What is split-brain and how do you prevent it?**
A: Both sites believing they're primary after a partition → divergent writes. Prevent with majority quorum (odd sites/witness), leases, and fencing the old primary before promotion.

**Q: Why do failovers fail in practice?**
A: Untested runbooks, insufficient standby capacity, hidden dependencies on the failed site, DNS TTL stragglers, and split-brain from ambiguous failure detection.

---

## 9. Key takeaways

1. **RTO/RPO** frame every DR conversation; each nine costs more.
2. Ladder: backup → pilot light → warm → active–passive → **active–active**.
3. Data replication is the hard part: **sync = RPO 0 + latency; async = fast + data loss; quorum across 3+ sites = both, at latency cost**.
4. **Split-brain** is the canonical failure; quorum + leases + fencing are the canonical cures.
5. **Unpracticed failover is fiction** — chaos drills and spare capacity are part of the design, not ops afterthoughts.

---

## 10. Papers & primary sources (storage-layer replication and failover)

These are the foundational papers behind the protocols and mechanisms in §4.

- **Chain Replication for Supporting High Throughput and Availability** — R. van Renesse, F. Schneider (OSDI 2004). The head→tail chain protocol; strong consistency with simple, deterministic node-failure recovery. <https://www.cs.cornell.edu/home/rvr/papers/OSDI04.pdf>
- **Object Storage on CRAQ: High-throughput Chain Replication for Read-Mostly Workloads** — J. Terrace, M. Freedman (USENIX ATC 2009). Adds clean/dirty versions so any chain node serves consistent reads. <https://www.usenix.org/legacy/event/usenix09/tech/full_papers/terrace/terrace.pdf>
- **Dynamo: Amazon's Highly Available Key-value Store** — DeCandia et al. (SOSP 2007). Sloppy quorums (W+R>N), hinted handoff, Merkle-tree anti-entropy, gossip membership — availability without a promotion step. <https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf>
- **In Search of an Understandable Consensus Algorithm (Raft)** — D. Ongaro, J. Ousterhout (USENIX ATC 2014). Leader election with the up-to-date-log restriction = lossless automatic failover; leases and terms/epochs. <https://raft.github.io/raft.pdf>
- **Spanner: Google's Globally-Distributed Database** — Corbett et al. (OSDI 2012). Paxos group per shard across ≥3 zones + TrueTime; replaces failover with continuous consensus. <https://research.google.com/archive/spanner-osdi2012.pdf>
- **Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases** — Verbitski et al. (SIGMOD 2017). "The log is the database": redo-log shipping to a 6-copy storage fleet (4/6 write, 3/6 read quorum) that self-heals without DB-side failover. <https://pages.cs.wisc.edu/~yxy/cs764-f20/papers/aurora-sigmod-17.pdf>
- **The Google File System** — Ghemawat, Gobioff, Leung (SOSP 2003). Primary-backup chunk replication with leases and a master that re-replicates lost chunks — the classic primary/backup storage failover reference. <https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf>
