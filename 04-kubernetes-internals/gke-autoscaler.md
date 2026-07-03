# GKE (Google Kubernetes Engine) Autoscaling

## 1. Background in 60 seconds

- **Kubernetes (k8s)**: orchestrates containers. You declare "run 5 replicas of this app, each needs 0.5 CPU + 1 GiB"; k8s finds machines (**nodes**) and runs the containers (**pods**) there.
- **Requests vs limits** (crucial vocabulary): a pod's **request** is the amount the scheduler *reserves* for it; the **limit** is the ceiling it may burst to. Scheduling and autoscaling decisions are based on **requests**, not actual usage — the root cause of most waste (see [../05-k8s-cost-optimization/](../05-k8s-cost-optimization/)).
- **GKE** = Google's managed Kubernetes: Google runs the control plane (API server, etcd, scheduler) and provides deep autoscaling automation.

**Autoscaling = matching capacity to load automatically**, in two directions (out/in = more/fewer copies; up/down = bigger/smaller copies) and at two levels (pods and nodes).

---

## 2. The four autoscalers (the mental map)

```
                     PODS                          NODES
             ┌─────────────────────┐      ┌─────────────────────────┐
 horizontal  │ HPA: more/fewer pod │      │ Cluster Autoscaler:     │
 (count)     │ replicas            │      │ more/fewer VMs          │
             ├─────────────────────┤      ├─────────────────────────┤
 vertical    │ VPA: bigger/smaller │      │ Node Auto-Provisioning: │
 (size)      │ CPU/mem per pod     │      │ create right-SIZED      │
             │                     │      │ node pools              │
             └─────────────────────┘      └─────────────────────────┘
```

They chain: HPA adds pods → pods don't fit → Cluster Autoscaler adds nodes. VPA right-sizes the pods so all the other math is honest.

### 2.1 HPA — Horizontal Pod Autoscaler
- Control loop (every ~15s): `desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)`.
- Example: target 60% CPU, current 90%, 4 replicas → `ceil(4 × 90/60) = 6`.
- Metrics: CPU/memory from the metrics server, plus **custom/external metrics** (requests/sec, Pub/Sub queue depth). Scaling on queue depth is often better than CPU for async workloads.
- Stabilization windows prevent flapping (default: scale up fast, scale down slowly over ~5 min).
- Best for **stateless** services; needs pods to have resource requests set.

### 2.2 VPA — Vertical Pod Autoscaler
- Watches actual usage history, recommends/sets better **requests** ("you asked for 2 CPU, you use 200m").
- Modes: `Off` (recommend only — safe default), `Initial` (apply at pod creation), `Auto` (apply live; historically required evicting/restarting the pod — newer k8s supports in-place resize).
- **Classic warning**: don't let HPA and VPA both act on CPU/memory for the same workload — they'll fight (HPA changes replica count based on utilization that VPA is simultaneously redefining).

### 2.3 Cluster Autoscaler (CA) — node horizontal
- Trigger to add: pods stuck **Pending** ("unschedulable" — nothing fits). CA simulates: "would this pod fit if I added a node from node pool X?" → grows the chosen pool (GKE spec: up to ~15k nodes).
- Trigger to remove: a node's pods all fit elsewhere and utilization is low (default threshold ~50% of requests) → drain (evict pods gracefully, respecting **PodDisruptionBudgets**) → delete.
- Note the design: CA reacts to *scheduling failure*, not CPU load — the load signal already got converted into pod count by HPA. Clean separation of concerns.
- Blockers it must respect: PDBs, pods with local storage, pods that can't move. These are why "why won't my cluster scale down?" is a daily ops question.

### 2.4 NAP — Node Auto-Provisioning
- CA picks among *existing* node pool shapes; **NAP creates new node pools** with machine shapes fitted to pending pods (e.g., a GPU pod arrives → NAP creates a GPU pool). Bin-packing with a dynamic set of bin sizes.

---

## 3. GKE Autopilot — the endgame

**Autopilot** mode removes nodes from your mental model entirely: you deploy pods; Google provisions and right-sizes underlying capacity; **you pay per pod resource request, not per VM**. It's GKE with all four autoscalers' jobs internalized by the platform + hardened defaults. Trade-off: less control (no privileged daemonsets, restricted node access), and per-pod pricing can exceed well-optimized standard clusters. Strategically, it's Google saying: capacity management should be the cloud's problem — the same thesis as the tools in [../05-k8s-cost-optimization/](../05-k8s-cost-optimization/).

---

## 4. The distributed-systems angle (for your interview)

Autoscaling is a **distributed control loop**, and it inherits classic control-theory/distributed problems:

1. **Feedback loops & oscillation**: scale-up increases capacity → utilization drops → scale-down → utilization rises… Damping = stabilization windows, cooldowns, hysteresis (same trick as failover health checks in [../03-reliability-failover/datacenter-failover.md](../03-reliability-failover/datacenter-failover.md)).
2. **Stale signals**: metrics pipelines have lag (~15–60s); decisions are made on the past. Aggressive reaction to stale data = overshoot.
3. **Interacting controllers**: HPA, VPA, CA are independent loops sharing state — coordination emerges through the API server's declarative state (level-triggered reconciliation), not through direct messaging. This "many small controllers reconciling declared state" is *the* Kubernetes design pattern.
4. **Bin packing** (CA/NAP) is NP-hard; production uses greedy heuristics + simulation.
5. **Cold-start latency**: a new VM takes ~1–2 min; scaling can't be purely reactive for spiky load → overprovisioning headroom, or *balloon pods* (low-priority placeholder pods that get preempted to make instant room — a neat trick worth mentioning).

---

## 5. Interview questions & strong answers

**Q: Walk me through what happens when traffic doubles.**
A: Load ↑ → metric crosses HPA target → HPA raises replicas → new pods go Pending → CA simulates fits, adds nodes (or NAP creates a fitting pool) → pods schedule → load spread. Reverse path on decline, slowly (stabilization + drain respecting PDBs).

**Q: Why did my cluster not scale down despite 20% utilization?**
A: Scale-down needs *evictable* pods: PDBs too tight, pods with local storage/no controller, system pods, or annotations block draining. Classic real-world answer.

**Q: HPA on CPU is failing for my queue consumer — why?**
A: CPU is a poor proxy for backlog; consumer may be I/O-bound while the queue grows. Scale on queue depth (external metric) instead.

**Q: Why do decisions use requests instead of actual usage?**
A: Requests are the scheduler's contract — predictable, gameable-but-stable. But it means chronic over-requesting creates invisible waste — which is precisely the business of CastAI/Kubecost/PerfectScale ([../05-k8s-cost-optimization/](../05-k8s-cost-optimization/)).

---

## 6. Key takeaways

1. Four autoscalers, one grid: **HPA/VPA (pods) × CA/NAP (nodes)**; horizontal = count, vertical = size.
2. The chain: **HPA → Pending pods → Cluster Autoscaler** — load signal becomes a scheduling signal.
3. Everything keys off **requests**, so wrong requests poison every layer.
4. It's a study in **distributed control loops**: feedback, damping, stale data, independent reconcilers.
5. **Autopilot** = Google absorbing the whole problem, billing per pod.
