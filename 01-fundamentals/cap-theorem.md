# CAP Theorem

## 1. The one-sentence version

> In a distributed system, when the network breaks (a *partition*), you must choose between giving **consistent** answers and staying **available** — you cannot have both.

Proposed by Eric Brewer (2000), formally proved by Gilbert & Lynch (2002).

---

## 2. Start with an analogy

Imagine two bank tellers, **Alice in Dhaka** and **Bob in New York**, who share one customer account. Normally they call each other after every transaction so both always know the balance.

Suddenly the **phone line goes dead** (this is the network partition). A customer walks up to Bob and asks to withdraw money. Bob has two choices:

1. **Refuse to serve the customer** until the phone line is back ("Sorry, I can't be sure of your balance right now"). → He chose **Consistency** over Availability.
2. **Serve the customer anyway** using his possibly-stale copy of the balance. → He chose **Availability** over Consistency (the two tellers may now disagree; the account might even go negative).

There is no third option. That is CAP.

---

## 3. The three letters, precisely

| Letter | Name | Precise meaning |
|---|---|---|
| **C** | Consistency | Every read returns the **most recent write** (or an error). Formally: *linearizability* — the system behaves as if there is only one copy of the data. |
| **A** | Availability | Every request to a **non-crashed node** gets a (non-error) response — even during a partition. |
| **P** | Partition tolerance | The system keeps functioning even when network messages between nodes are lost or delayed. |

### The most common misunderstanding
People say "pick 2 of 3." That's misleading. **In a real distributed system, partitions WILL happen** — cables get cut, switches fail, garbage-collection pauses make a node look dead. So **P is not optional**. The real choice is:

> **When a partition happens, do you sacrifice C or A?**

- **CP system**: during a partition, some nodes refuse requests (lose availability) to stay correct.
- **AP system**: during a partition, all nodes keep answering, but answers may be stale or conflicting.
- **CA** only exists on a single machine (no partitions possible) — which is not a distributed system.

---

## 4. Real systems classified

| System | Choice | How it behaves during a partition |
|---|---|---|
| **etcd / ZooKeeper / Consul** | CP | Minority side of the partition stops serving writes (Raft/Paxos needs a majority). |
| **Google Spanner** | CP (effectively) | Uses Paxos + TrueTime atomic clocks; Google argues partitions are so rare on its network that it *feels* CA. |
| **HBase, MongoDB (default)** | CP-leaning | Primary steps down if it can't reach a majority. |
| **Cassandra, DynamoDB** | AP | Any replica accepts writes; conflicts fixed later ("eventual consistency"). |
| **DNS** | AP | Massively available, famously stale. |
| **A single PostgreSQL server** | "CA" | No partition possible — but also not distributed. |

---

## 5. Eventual consistency (the AP world)

If you give up strong consistency, you promise something weaker:

> **Eventual consistency**: if writes stop, all replicas *eventually* converge to the same value.

Techniques used to converge:
- **Last-writer-wins (LWW)** — timestamps decide; simple but can silently drop updates.
- **Vector clocks** — track causality, detect true conflicts (used by Amazon Dynamo).
- **CRDTs** (Conflict-free Replicated Data Types) — data structures mathematically guaranteed to merge (e.g., counters, sets used in Redis and Riak).
- **Read repair / anti-entropy (Merkle trees)** — background processes that compare and heal replicas.

Tunable consistency (Cassandra style): with **N** replicas, require **W** acks per write and **R** replicas per read. If **R + W > N**, reads intersect writes → strong-ish consistency. Choose R=1, W=1 → fast but weak.

---

## 6. Beyond CAP: PACELC (know this — it impresses interviewers)

CAP only talks about what happens *during* a partition. **PACELC** (Daniel Abadi, 2012) completes the picture:

> **If Partition → choose Availability or Consistency; Else (normal operation) → choose Latency or Consistency.**

Even with a perfect network, making a write consistent requires waiting for replicas to acknowledge → **higher latency**. Examples:

- **DynamoDB**: PA/EL — available during partitions, fast (weaker consistency) normally.
- **Spanner**: PC/EC — consistent always, pays latency (cross-region Paxos commits).
- **MongoDB**: PA/EC (roughly).

---

## 7. Common interview questions & good answers

**Q: Why can't we have all three?**
A: Suppose nodes N1 and N2 are partitioned. A write goes to N1. A read arrives at N2. N2 either answers (available but possibly stale → not consistent) or refuses/waits (consistent but not available). This is the essence of the Gilbert–Lynch proof.

**Q: Is CAP a reason to always pick eventual consistency?**
A: No. Partitions are rare in well-built data centers; sacrificing consistency 100% of the time to handle a 0.01%-of-the-time event is a poor trade. This is Spanner's philosophy.

**Q: What does "consistency" in CAP have to do with ACID's C?**
A: Nothing — a classic trap. ACID's C means database invariants hold after a transaction. CAP's C means linearizability across replicas.

**Q: How does Raft relate to CAP?**
A: Raft is a CP protocol: it guarantees consistency by requiring a majority quorum; the minority side of a partition becomes unavailable for writes. (See [raft-and-paxos.md](raft-and-paxos.md).)

---

## 8. Key takeaways (memorize these)

1. P is mandatory; the real trade is **C vs A during a partition**.
2. CAP-Consistency = **linearizability** ("acts like one copy").
3. AP systems repay the debt with **eventual consistency + conflict resolution**.
4. **PACELC** adds the everyday trade-off: consistency costs **latency** even without partitions.
5. Real systems are on a **spectrum** (tunable quorums), not in three neat boxes.
