# Distributed Systems Study Notes

Beginner-friendly but detailed notes for PhD-interview preparation. Each note assumes zero prior knowledge, builds up with analogies, then goes deep enough for technical discussion, and ends with likely interview questions + key takeaways.

## Directory map

### `01-fundamentals/` — the theory everything else stands on
- [CAP Theorem](01-fundamentals/cap-theorem.md) — consistency vs availability under partitions; PACELC
- [Raft & Paxos](01-fundamentals/raft-and-paxos.md) — distributed consensus, leader election, replicated logs

### `02-distributed-storage/` — storing data across machines
- [Distributed Storage](02-distributed-storage/distributed-storage.md) — partitioning, replication, erasure coding, GFS/HDFS
- [Google Object Storage Design](02-distributed-storage/google-object-storage.md) — GCS, Colossus, Spanner-backed metadata
- [Time Series Databases](02-distributed-storage/time-series-database.md) — LSM trees, Gorilla compression, cardinality

### `03-reliability-failover/` — surviving disasters
- [Data Center Failover](03-reliability-failover/datacenter-failover.md) — RTO/RPO, replication topologies, split-brain, chaos engineering

### `04-kubernetes-internals/` — coordination in practice
- [GKE Autoscaler](04-kubernetes-internals/gke-autoscaler.md) — HPA, VPA, Cluster Autoscaler, NAP, Autopilot
- [Leases in Kubernetes](04-kubernetes-internals/kubernetes-lease.md) — leader election, heartbeats, fencing

### `05-k8s-cost-optimization/` — the resource-efficiency industry
- [Cast AI](05-k8s-cost-optimization/cast-ai.md) — automated bin packing, spot automation, rightsizing
- [Kubecost](05-k8s-cost-optimization/kubecost.md) — cost allocation, showback, FinOps
- [PerfectScale](05-k8s-cost-optimization/perfectscale.md) — ML-driven workload optimization + resilience

## Suggested study order

1. **CAP Theorem** → 2. **Raft & Paxos** (the two theory pillars — read these first; every other note references them)
3. **Distributed Storage** → 4. **Google Object Storage** → 5. **Time Series DB**
6. **Data Center Failover** (applies consensus + replication to disasters)
7. **Kubernetes Lease** (consensus applied in k8s) → 8. **GKE Autoscaler**
9. **Kubecost** → 10. **Cast AI** → 11. **PerfectScale** (read as one story: see → automate infra → tune workloads)

## Recurring themes to weave into interview answers

- **Partition + replicate** explains almost every storage system.
- **Separate control plane from data plane** (GFS, Colossus, GCS, k8s, TSDBs) — the universal scalability trick.
- **Quorums (2f+1)** buy safety; **leases** buy practical failure detection; **fencing** closes the last gap.
- **Everything is a control loop**: autoscalers, repair jobs, optimizers — observe → decide → act, with damping.
- **Every guarantee has a price** (CAP/PACELC): consistency costs latency; durability costs storage; availability costs capacity.

## Papers worth name-dropping

- Lamport — *The Part-Time Parliament* (Paxos, 1998) / *Paxos Made Simple* (2001)
- Ongaro & Ousterhout — *In Search of an Understandable Consensus Algorithm* (Raft, 2014)
- Ghemawat et al. — *The Google File System* (2003)
- Corbett et al. — *Spanner: Google's Globally-Distributed Database* (2012)
- Pelkonen et al. — *Gorilla: A Fast, Scalable, In-Memory Time Series Database* (2015)
- Rzadca et al. — *Autopilot: Workload Autoscaling at Google* (EuroSys 2020)
- Gilbert & Lynch — *Brewer's Conjecture and the Feasibility of CAP* (2002)
