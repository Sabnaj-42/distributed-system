# PerfectScale — AI-Powered Kubernetes Workload Optimization

## 1. Where it fits

Third of the trio (read [kubecost.md](kubecost.md) and [cast-ai.md](cast-ai.md) first):

| Tool | Center of gravity | Level |
|---|---|---|
| Kubecost | **See** costs (allocation, showback) | Cluster accounting |
| Cast AI | **Automate infrastructure** (nodes, bin packing, spot) | Node/infra level |
| **PerfectScale** (founded 2021, Israel; **acquired by DoiT, 2024**) | **Automate workload sizing** (requests, limits, HPA) with equal weight on *reliability* | Pod/workload level |

PerfectScale's pitch: rightsizing done naïvely *causes outages* (cut memory requests too far → OOM kills; CPU too far → throttling). So optimize **cost and resilience as a joint objective**, using ML over usage patterns, and automate the boring-but-scary act of continuously updating every workload's resources.

---

## 2. What it actually does

Agent in your cluster collects per-container usage time series (again: a TSDB pipeline — [../02-distributed-storage/time-series-database.md](../02-distributed-storage/time-series-database.md)); the platform then:

### 2.1 Workload rightsizing (the core)
- Analyzes weeks of usage per container: percentiles, daily/weekly seasonality, spike shapes, OOM/throttle events.
- Recommends **requests and limits** per container — not from a single utilization average, but fitted to the pattern (e.g., memory sized to observed peaks + safety margin because memory overrun = kill; CPU sized closer to typical because overrun = mere throttling). **The asymmetry between CPU (compressible) and memory (incompressible) resources is the key domain insight.**
- **Automation mode ("Optimize")**: applies changes continuously and safely (respecting deploy windows, gradual rollout) instead of dumping YAML suggestions on engineers. Essentially a production-hardened, ML-driven VPA ([../04-kubernetes-internals/gke-autoscaler.md](../04-kubernetes-internals/gke-autoscaler.md)).

### 2.2 Resilience surface (the differentiator)
- Detects **under-provisioning** risks with the same seriousness as waste: containers flirting with OOM, CPU-throttled pods, missing limits, HPA misconfigurations, single-replica deployments.
- Scores/prioritizes issues so platform teams work a ranked queue rather than a wall of dashboards.
- This dual framing — "we reduce cost *and* prevent the incidents that naive cost-cutting causes" — is the product's identity.

### 2.3 Horizontal + vertical together
- Also tunes **HPA parameters** (target utilization, min/max replicas) using the fixed vertical sizes — addressing the classic "HPA and VPA fight" problem by having one brain set both ([gke-autoscaler note, §2.2](../04-kubernetes-internals/gke-autoscaler.md)).

---

## 3. Why "AI-powered" is (mostly) legitimate here

Strip the marketing; the real ML problems are:
1. **Time series forecasting**: predict a container's resource needs from seasonal, trending, spiky history — set requests for *tomorrow's* load, not yesterday's average.
2. **Quantile/risk estimation**: choosing memory = "P99.9 of predicted usage + margin" is explicitly a tail-risk problem; the cost of under-estimating (outage) vastly exceeds over-estimating (a few dollars) → asymmetric loss functions.
3. **Multi-objective optimization**: minimize $ subject to SLO-violation probability < ε, per workload class.
4. **Feedback loops**: applied recommendations change the observed data (autoscalers react, packing changes) — the system must learn under its own influence. Genuine control-theory-meets-ML territory, and a nice thing to mention to a professor: it's **closed-loop learning**, not offline prediction.

---

## 4. The bigger pattern (say this in your interview)

All three tools are symptoms of one trend: **declarative infrastructure created a new optimization layer**. Kubernetes lets humans declare resource needs; humans are terrible at it; so a market emerged of *controllers that write the declarations for you*, closing the loop with telemetry. The intellectual lineage:

```
Google Borg/Autopilot (internal, ~2020 paper: ML-set resource limits at Google)
        → open-source k8s VPA (crude)
        → commercial closed-loop optimizers (Cast AI, PerfectScale, StormForge…)
        → cloud platforms absorbing it (GKE Autopilot)
```

Google's **Autopilot paper (EuroSys 2020)** is the academic anchor: ML-driven vertical scaling at Google cut slack from ~46% to ~23% across the fleet while *reducing* OOMs. If you cite one thing to a professor, cite that — the commercial tools are that paper productized for everyone else.

---

## 5. Honest critique

- Overlap is real: Cast AI does rightsizing too; Kubecost recommends sizes; VPA is free. The differentiators are automation safety, the resilience framing, and ML quality — hard to verify from outside.
- Closed SaaS: your utilization telemetry leaves the cluster; ML claims are unauditable.
- Same Goodhart risk as all optimizers: optimizing to observed history under-provisions for the unprecedented spike (launch day, incident retry storms). Human-set floors remain necessary.

---

## 6. Key takeaways

1. PerfectScale = **continuous, ML-driven rightsizing of pod requests/limits + HPA tuning**, with **resilience as a co-equal objective**.
2. Key technical insight: **CPU is compressible (throttle), memory is incompressible (OOM kill)** → size them with different risk tolerances.
3. The ML substance = forecasting + tail-quantile estimation + closed-loop control, not buzzwords.
4. Trio summary: **Kubecost sees, Cast AI repacks infrastructure, PerfectScale tunes workloads** — converging feature sets, different centers of gravity.
5. Academic anchor: **Google's Autopilot (EuroSys 2020)** — the origin of this whole product category.
