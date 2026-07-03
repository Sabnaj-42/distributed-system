# Raft (and Paxos): Distributed Consensus

> Note: you may see "Paxton" written informally — the correct name is **Paxos**, from Leslie Lamport's paper *The Part-Time Parliament* (named after the Greek island Paxos).

## 1. The problem: consensus

**Consensus** = getting a group of machines to agree on a value (or a sequence of values) even when some machines crash and the network drops messages.

Why do we need it? Because a single server fails. To survive failures we keep **replicas**, and replicas must agree on the order of operations, otherwise their data diverges. Consensus is the foundation of:

- Leader election ("who is the primary database?")
- Replicated logs (every replica applies the same commands in the same order → **state machine replication**)
- Configuration stores: **etcd** (Raft), **ZooKeeper** (ZAB, Paxos-like), **Chubby** at Google (Paxos)
- Google Spanner, CockroachDB, TiDB — every data shard is a Paxos/Raft group

**Key safety requirement:** never let two nodes think different values were "decided." Availability can wait; disagreement is forever.

**Fundamental limits worth citing in an interview:**
- **FLP impossibility (1985)**: in a fully asynchronous system, no deterministic algorithm guarantees consensus if even one node can crash. Practical systems dodge this with timeouts (partial synchrony) and randomness.
- **Quorum math**: to tolerate **f** crash failures you need **2f+1** nodes (majority = f+1). 3 nodes tolerate 1 failure, 5 tolerate 2.

---

## 2. Paxos in a nutshell (know the shape, not every detail)

Paxos decides **one value** using two phases; participants are *proposers*, *acceptors*, *learners*:

1. **Phase 1 (Prepare/Promise)**: a proposer picks a proposal number `n` and asks a majority of acceptors to *promise* to ignore anything numbered below `n`. Acceptors reply with any value they already accepted.
2. **Phase 2 (Accept/Accepted)**: the proposer sends its value (or the highest-numbered value it learned in phase 1 — this rule is the heart of safety) to a majority. If a majority accepts, the value is **chosen** forever.

- **Multi-Paxos**: run Paxos repeatedly for a log; optimize by electing a stable leader so phase 1 is done once, then each log entry needs only phase 2. At this point Multi-Paxos looks a lot like Raft.
- Reputation: **correct but notoriously hard to understand and implement**. Google's "Paxos Made Live" paper documents how painful productionizing it was. This pain is literally why Raft exists.

---

## 3. Raft — consensus designed for understandability

Diego Ongaro & John Ousterhout, 2014: *In Search of an Understandable Consensus Algorithm*. Raft is equivalent in power to Multi-Paxos but decomposes the problem into three understandable pieces:

### 3.1 The setup

- A cluster of (usually) 3 or 5 servers. Each server is a **Follower**, **Candidate**, or **Leader**.
- Time is divided into **terms** (monotonically increasing integers). Each term has at most one leader. Terms act as a logical clock: any message from an older term is rejected; seeing a newer term makes you step down to follower.
- Each server keeps a **log** of commands. The goal: identical logs on all servers.

```
            times out,                receives majority
            starts election           of votes
 Follower ────────────────► Candidate ────────────────► Leader
    ▲                          │                           │
    └──────────────────────────┴───────────────────────────┘
        discovers current leader or a higher term
```

### 3.2 Piece 1 — Leader election

- Followers expect periodic **heartbeats** (empty `AppendEntries` RPCs) from the leader.
- If a follower hears nothing for a **randomized election timeout** (e.g., 150–300 ms), it increments the term, becomes a **candidate**, votes for itself, and sends `RequestVote` to everyone.
- A server grants its (single) vote per term, but **only to a candidate whose log is at least as up-to-date as its own** — this is how Raft guarantees the new leader already has all committed entries.
- **Majority of votes → leader.** Split vote → timeout → retry with a new term. The *randomized* timeouts make repeated splits improbable — an elegantly simple fix.

### 3.3 Piece 2 — Log replication

1. Client sends a command to the leader.
2. Leader appends it to its own log as entry `(term, index, command)`.
3. Leader sends `AppendEntries` to all followers. Each RPC carries the previous entry's `(index, term)`; a follower rejects the append if its log doesn't match at that spot (**consistency check**), and the leader backs up and retransmits until logs converge. Conflicting follower entries get overwritten — **the leader's log is the truth; logs flow only leader → follower**.
4. Once a **majority** has stored the entry, the leader marks it **committed**, applies it to its state machine, and answers the client. Followers learn the commit index from later heartbeats and apply too.

**Key safety property (Leader Completeness):** a committed entry is on a majority; any future leader must win a majority of votes; the up-to-date-log voting rule guarantees those two majorities intersect at a node that forces the winner to have the entry. Therefore **committed entries are never lost**.

Subtlety worth knowing: a leader never counts replicas of an *old-term* entry to declare it committed (Figure 8 of the paper); it commits old entries only indirectly, by committing an entry from its **own** term.

### 3.4 Piece 3 — Safety odds & ends

- **Membership changes**: done via **joint consensus** (or the simpler single-server-change method) so there's never a moment when two disjoint majorities could elect two leaders.
- **Log compaction / snapshots**: logs can't grow forever; servers snapshot the state machine and truncate the log; slow followers receive `InstallSnapshot`.
- **Client semantics**: exactly-once is achieved by giving clients session IDs + sequence numbers so retried commands aren't applied twice.

### 3.5 What happens during a partition? (ties to CAP)

5-node cluster splits 3 / 2:
- The 3-node side can elect a leader and commit → **stays available**.
- The 2-node side has no majority → its leader (if any) can't commit anything; a candidate can't win. **Unavailable but safe.**
- On heal, the minority side sees the higher term, steps down, and its uncommitted entries are overwritten. **Raft is a CP system** (see [cap-theorem.md](cap-theorem.md)).

---

## 4. Raft vs Paxos — the comparison you'll be asked

| Aspect | Paxos (Multi-Paxos) | Raft |
|---|---|---|
| Primary goal | Theoretical minimality | **Understandability** |
| Leader | Optimization, loosely specified | Built-in, central to the design |
| Log | Gaps/holes allowed | Strictly sequential, no holes |
| Who wins election | Anyone (then fetches missing entries) | Only a node with an up-to-date log |
| Spec completeness | Famously incomplete for practice | Full recipe: election, replication, membership, snapshots |
| Used in | Chubby, Spanner, Megastore | etcd, Consul, CockroachDB, TiKV, RabbitMQ streams, Kafka KRaft |
| Flexibility | More room for exotic variants (Flexible Paxos, EPaxos) | More prescriptive |

Honest expert take: Raft ≈ Multi-Paxos with specific engineering choices baked in. Raft won mindshare because engineers can actually implement it from the paper.

---

## 5. Interview questions & strong answers

**Q: Why odd cluster sizes (3, 5)?**
A: Majorities. 4 nodes still tolerate only 1 failure (majority = 3) — same fault tolerance as 3 nodes but more machines and slower quorums. Even sizes buy nothing.

**Q: Can Raft lose committed data?**
A: No — committed means on a majority, and election rules force any new leader to hold every committed entry. Uncommitted (client not yet acked) entries can be lost, which is fine.

**Q: Two leaders at once?**
A: Two nodes can *believe* they're leaders, but in different terms. The stale leader can't commit (no majority will follow an old term) and steps down on first contact with the newer term. Safety holds.

**Q: Why randomized election timeouts?**
A: To break symmetry. If all followers timed out simultaneously they'd perpetually split the vote. Randomization makes one candidate usually start (and win) first.

**Q: Where does Kubernetes use this?**
A: The entire cluster state lives in **etcd**, a Raft-backed key-value store. Also, k8s controllers do leader election via **Lease objects** — a lighter mechanism built *on top of* etcd's consensus (see [../04-kubernetes-internals/kubernetes-lease.md](../04-kubernetes-internals/kubernetes-lease.md)).

---

## 6. Key takeaways

1. Consensus = agreeing on a **replicated log**; it powers leader election and fault-tolerant storage.
2. **2f+1** nodes tolerate **f** failures; everything is majority quorums.
3. Raft = elect a leader with the most complete log → leader dictates the log → commit at majority.
4. Terms + voting rules give the core guarantee: **committed entries survive any legal failure**.
5. Paxos and Raft are equivalent in power; Raft is the one you can implement from the paper.
