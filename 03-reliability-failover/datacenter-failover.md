# Data Center Failover

## 1. The problem

Your entire application runs in one data center (DC). Then: a fiber cut, a power failure, a flood, a bad config push (most common!), or a cloud region outage. **Failover** = detecting that a site is unhealthy and shifting traffic and data to another site so users barely notice.

Two goals, two acronyms you must know:

| Term | Meaning | Question it answers |
|---|---|---|
| **RTO** — Recovery Time Objective | Max acceptable downtime | "How long until we're back up?" |
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

## 4. Traffic failover — how requests move to DC-B

- **DNS-based** (Route 53, Cloud DNS + health checks): change the name → new IPs. Simple, global; but DNS caching/TTLs mean minutes of stragglers.
- **Anycast / global load balancers** (Google GLB, Cloudflare): one IP announced from many sites via BGP; failover is near-instant routing change, no client cooperation needed.
- **Health checking** drives all of it: probes must measure *user-visible* health (can I complete a real request?) not just "is the port open." Flapping protection: fail fast, recover slowly (hysteresis).

Failover sequencing matters: **fence old primary → promote data replica → verify → shift traffic**. Shifting traffic before the data layer is ready just moves the outage.

---

## 5. Hard-won operational lessons (interviewers love these)

1. **Untested failover doesn't exist.** DR that's never exercised fails when needed. Hence **chaos engineering**: Netflix Chaos Monkey (kill instances), Chaos Kong (evacuate an entire AWS region regularly, in production); Google's DiRT exercises.
2. **Capacity**: surviving DCs must absorb the failed DC's load. Run every site ≤ N-1/N capacity (e.g., 2 DCs → each ≤50% utilized), or failover triggers a cascading overload — the second outage is self-inflicted.
3. **The failover system itself must be simpler and more reliable than what it protects.** Control planes causing outages is a recurring postmortem theme.
4. **Correlated failures**: the disaster and its response can share a dependency (auth service, DNS, config store in the failed region). Facebook's 2021 BGP outage locked engineers out of the tools needed to fix it.
5. **Failback** (returning to DC-A) is a second migration and needs the same rigor; data written in DC-B must reconcile back.
6. **Practice regularly**: Google SRE's rule of thumb — if you haven't failed over recently, assume you can't.

---

## 6. Case study to cite: Google Spanner's approach
Instead of primary/standby failover, Spanner makes every data shard a **Paxos group across ≥3 zones/regions**. A zone loss is absorbed by the quorum with zero data loss and ~no downtime — "failover" is replaced by *continuous consensus*. Trade-off: cross-site commit latency on every write (the PACELC latency cost). This is the intellectual endpoint of the topology ladder.

---

## 7. Interview questions & strong answers

**Q: Design DR for a payment system.**
A: Payments ⇒ RPO=0 ⇒ synchronous or quorum replication across ≥3 AZs; cross-region async copy with an explicit, small RPO for region loss (or 3-region quorum for RPO=0 at latency cost); fencing + witness against split-brain; regular game-day drills; N+1 capacity.

**Q: Sync vs async replication?**
A: RPO=0 vs write latency — a direct CAP/PACELC instance. Sync within a region (1 ms), async or quorum across regions, chosen per data class.

**Q: What is split-brain and how do you prevent it?**
A: Both sites believing they're primary after a partition → divergent writes. Prevent with majority quorum (odd sites/witness), leases, and fencing the old primary before promotion.

**Q: Why do failovers fail in practice?**
A: Untested runbooks, insufficient standby capacity, hidden dependencies on the failed site, DNS TTL stragglers, and split-brain from ambiguous failure detection.

---

## 8. Key takeaways

1. **RTO/RPO** frame every DR conversation; each nine costs more.
2. Ladder: backup → pilot light → warm → active–passive → **active–active**.
3. Data replication is the hard part: **sync = RPO 0 + latency; async = fast + data loss; quorum across 3+ sites = both, at latency cost**.
4. **Split-brain** is the canonical failure; quorum + leases + fencing are the canonical cures.
5. **Unpracticed failover is fiction** — chaos drills and spare capacity are part of the design, not ops afterthoughts.
