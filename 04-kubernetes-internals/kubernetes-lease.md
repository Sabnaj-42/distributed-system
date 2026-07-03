# Leases in Kubernetes

## 1. The general concept first: what is a lease?

A **lease** = a lock with an expiration time.

Plain locks are dangerous in distributed systems: if the lock holder crashes, the lock is held forever. A lease fixes this: *"You hold this for 15 seconds. Renew before it expires or you lose it."* If the holder dies, the lease simply expires and someone else takes over. **The timeout converts 'crashed vs slow' ambiguity into a bounded wait.**

Leases appear throughout distributed systems: GFS chunk leases, DNS TTLs, DHCP, cache coherence, primary election in databases, fencing in failover ([../03-reliability-failover/datacenter-failover.md](../03-reliability-failover/datacenter-failover.md)). Kubernetes made them a first-class API object.

---

## 2. The Lease object in Kubernetes

A tiny record in the API group `coordination.k8s.io/v1`, stored (like everything) in **etcd**:

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: my-controller-leader
  namespace: kube-system
spec:
  holderIdentity: "controller-pod-abc123"   # who holds it
  leaseDurationSeconds: 15                  # validity window
  acquireTime: "2026-07-03T10:00:00Z"
  renewTime: "2026-07-03T10:00:10Z"         # heartbeat timestamp
  leaseTransitions: 3                       # how many times leadership changed
```

The mechanics rest on two layers of guarantees:
1. **etcd's Raft consensus** ([../01-fundamentals/raft-and-paxos.md](../01-fundamentals/raft-and-paxos.md)) makes the record itself consistent and durable.
2. **Optimistic concurrency control**: every k8s object has a `resourceVersion`; an update fails if the object changed since you read it (a compare-and-swap). So two contenders racing to grab an expired lease → exactly one write wins, the other gets a 409 Conflict and backs off. No extra locking protocol needed.

---

## 3. Use #1 — Leader election (active–passive HA)

Many k8s components must be HA but **only one instance may act at a time** (otherwise two schedulers assign the same pod to different nodes). Components that use this: `kube-scheduler`, `kube-controller-manager`, `cloud-controller-manager`, cluster-autoscaler, and most operators (via `client-go`'s `leaderelection` package or controller-runtime).

The algorithm each replica runs:

```
loop:
  read Lease
  if lease expired (renewTime + duration < now) or I already hold it:
      try compare-and-swap myself in as holderIdentity
      if write succeeded → I am the leader; run the control loop; renew every ~10s
  else:
      stand by, retry after a bit
if I am leader and renewal fails repeatedly → STOP ACTING IMMEDIATELY, step down
```

Timing knobs (defaults): `leaseDuration` 15s, `renewDeadline` 10s, `retryPeriod` 2s. Failover time on leader crash ≈ lease duration (~15s of leaderless-but-safe standstill).

### The honest caveat (great interview material)
This gives **at-most-one leader *almost* always**, but not absolutely: a leader hit by a long GC pause or clock skew may keep acting for a moment after its lease expired and a new leader took over. K8s leader election relies on the holder *voluntarily* checking its own lease. Rigorous systems add **fencing tokens**: use `leaseTransitions` (a monotonically increasing epoch) on every downstream write so a stale leader's writes are rejected. This is exactly Martin Kleppmann's famous "How to do distributed locking" critique — citing it signals depth.

---

## 4. Use #2 — Node heartbeats (failure detection)

Every kubelet maintains a Lease named after its node in the `kube-node-lease` namespace, renewing every **10s**. If the node-lifecycle-controller sees no renewal within **40s** → node marked `NotReady` → after eviction timeout, pods are rescheduled elsewhere.

Why Leases instead of the old method? Pre-1.14, kubelets heartbeat by updating their whole (large) Node object every 10s; at 5,000 nodes this hammered etcd (every update = a Raft commit + watch fan-out). The Lease object is tiny and watched by almost nobody → **heartbeats became ~100× cheaper, unlocking 5k-node clusters**. Lovely scalability story: *separate the high-frequency small signal (liveness) from the low-frequency big one (node status)* — control/data-plane separation again.

Also in the family: **API server identity leases** (`kube-apiserver-<hash>`, so components can discover live API servers) and coordinated storage-version migration.

---

## 5. The distributed-systems lens (what this topic is really about)

1. **Failure detection is impossible to do perfectly** (can't distinguish crashed/slow/partitioned); leases make it *practical* by bounding uncertainty with time.
2. **Leases trade availability for safety**: on expiry+partition, the old holder stops (unavailable) rather than risk two actors (inconsistent) — a mini CAP decision ([../01-fundamentals/cap-theorem.md](../01-fundamentals/cap-theorem.md)).
3. **They depend on clocks** — only on *rates* (measuring 15 seconds locally), not synchronized wall time; still, a paused process (GC, VM migration) breaks the assumption → fencing tokens as the backstop.
4. **Layering**: heavyweight consensus (Raft in etcd) at the bottom, cheap CAS-based leases on top, thousands of controllers coordinating through them. You don't run Paxos per controller; you *rent* consistency from etcd.
5. It's the k8s instance of a universal pattern: GFS gives chunkservers 60s leases; DHCP leases IPs; DNS TTLs lease cache validity.

---

## 6. Interview questions & strong answers

**Q: How does Kubernetes ensure only one scheduler is active?**
A: Replicas race to CAS their identity into a Lease object; etcd's consistency + resourceVersion CAS makes exactly one winner; holder renews every 10s and must step down if renewals fail; others take over ~15s after the holder dies.

**Q: Can two leaders exist briefly?**
A: Yes — pause/partition can leave a stale leader acting past expiry. Mitigation: leader self-checks before acting, short lease durations, and fencing tokens (epoch numbers) validated by downstream systems.

**Q: Why did node heartbeats move to Lease objects?**
A: Full Node-object updates were etcd-expensive (large writes + watch fan-out at high frequency). Tiny dedicated Lease objects cut the cost ~100×, enabling large clusters.

**Q: What happens to leases during an etcd partition?**
A: Leases live in etcd, so the minority side can't renew or acquire (writes need Raft majority) — leadership actions safely stall rather than fork. CP behavior end-to-end.

---

## 7. Key takeaways

1. Lease = **lock + TTL**: crash-safe mutual exclusion via time.
2. Two big k8s uses: **leader election** (one active controller) and **node heartbeats** (cheap failure detection).
3. Correctness comes from **etcd Raft + resourceVersion compare-and-swap** — leases rent consistency from consensus.
4. Know the caveat: **at-most-one-leader is probabilistic**; fencing tokens make it rigorous.
5. The heartbeat redesign is a textbook lesson: **make the frequent signal small**.
