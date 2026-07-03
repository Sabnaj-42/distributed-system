# Cast AI — Automated Kubernetes Resource Optimization

## 1. The problem all three tools in this folder attack

Kubernetes clusters are chronically wasteful. Industry reports (CNCF/Datadog/Cast AI's own surveys) consistently find **only ~10–15% of provisioned CPU is actually used**. Why:

1. **Over-requesting**: developers set pod CPU/memory *requests* defensively ("ask for 4 CPU just in case"); the scheduler reserves it; it sits idle. (Requests vs limits: see [../04-kubernetes-internals/gke-autoscaler.md](../04-kubernetes-internals/gke-autoscaler.md).)
2. **Wrong-sized nodes**: pods don't tile perfectly onto VM shapes → stranded capacity (bin-packing loss).
3. **Static provisioning for peak**: capacity bought for Black Friday runs all year.
4. **Expensive capacity types**: on-demand VMs when spot/preemptible (60–90% cheaper) would do.

Cast AI's positioning: don't just *report* this waste (Kubecost's origin, [kubecost.md](kubecost.md)) or *recommend* fixes (PerfectScale's origin, [perfectscale.md](perfectscale.md)) — **take over and fix it continuously, automatically**.

---

## 2. What Cast AI does (the feature anatomy)

Cast AI is a SaaS platform + an agent you install in your cluster (EKS/GKE/AKS). Read-only agent first ("here's what you'd save"), then a cluster controller with write permissions to act. Main engines:

### 2.1 Autoscaler + bin-packing (the core)
Replaces/augments the stock Cluster Autoscaler:
- Continuously computes the **cheapest set of VMs that fits current pods** — full-dimensional bin packing (CPU, memory, GPU) across *all* instance families and sizes, not a fixed node-pool list.
- **Rebalancing**: periodically re-solves the packing and migrates pods onto a cheaper node set (drain + reschedule), e.g., consolidating 10 half-empty nodes into 5 full ones. The stock CA only removes nodes that are *already* nearly empty; Cast AI actively *creates* that emptiness.
- Removes the node-pool abstraction: it picks arbitrary instance types per need (similar spirit to GKE NAP / Karpenter).

### 2.2 Spot instance automation
Spot/preemptible VMs are cheap because the cloud can reclaim them with ~30–120s notice — fine for stateless/interruptible pods, if someone handles the churn:
- Predicts interruption risk per instance type, diversifies across types/zones.
- On reclaim notice: drains the node, pre-provisions replacement capacity ("fallback" to on-demand if spot is unavailable, back to spot when it returns).
- Lets you mark workloads spot-safe vs on-demand-only.

### 2.3 Workload rightsizing (vertical)
Watches actual usage, then **automatically adjusts pod requests** (a productionized VPA) so the bin-packing math operates on truth instead of developer guesses. Requests shrink → pods pack denser → the autoscaler deletes freed nodes → the savings become real. **Key insight: rightsizing and bin-packing compound — request fixes without node consolidation save nothing.**

### 2.4 Supporting cast
Cost monitoring/allocation (per cluster/namespace/workload), commitment (CUD/RI) planning, GPU sharing, and a security posture module.

---

## 3. Why this is a distributed-systems topic (your interview angle)

1. **Online bin packing under uncertainty**: pods arrive/leave continuously, instance prices and spot availability change hourly → NP-hard packing solved with greedy/heuristic re-optimization loops. Classic scheduling-theory-meets-production.
2. **Migration safety**: rebalancing means evicting live pods — must respect PodDisruptionBudgets, graceful termination, stateful constraints; effectively a continuous, tiny version of data-center failover ([../03-reliability-failover/datacenter-failover.md](../03-reliability-failover/datacenter-failover.md)).
3. **Control loop layering**: Cast AI's controller sits *above* Kubernetes' own reconcilers, using the same declarative pattern — observe (metrics), decide (optimizer), act (k8s API) — another instance of the control-loop stack from [../04-kubernetes-internals/gke-autoscaler.md](../04-kubernetes-internals/gke-autoscaler.md).
4. **Risk economics**: spot automation converts "cheap but unreliable capacity" into "cheap and reliable-enough" by diversification + fast reaction — reliability engineered from unreliable parts, the founding idea of distributed systems.

---

## 4. Honest critique (shows maturity in interviews)

- **Trust**: you hand cluster-mutation rights to a third-party SaaS — blast-radius and security questions are legitimate.
- **Aggressive consolidation vs stability**: every rebalance is deliberate disruption; latency-sensitive or poorly-drain-tolerant apps can suffer. There's an inherent **cost vs churn** trade-off dial.
- Savings claims (often "50%+") are measured against *badly configured* baselines; a well-tuned cluster saves less.
- Overlaps with free/open alternatives: **Karpenter** (AWS's open-source bin-packing node autoscaler) covers a chunk of 2.1.

---

## 5. Key takeaways

1. K8s waste = over-requesting + bad bin packing + peak provisioning + on-demand-only capacity.
2. Cast AI = **closed-loop automation**: rightsize requests + repack nodes + surf spot markets, continuously.
3. The two levers **compound**: request truth × dense packing × cheap capacity.
4. Technically it's **online bin packing + safe live migration + layered control loops**.
5. Positioning vs siblings: Kubecost *observes*, PerfectScale *recommends-then-automates*, **Cast AI automates hardest** ([kubecost.md](kubecost.md), [perfectscale.md](perfectscale.md)).
