# Kubecost — Kubernetes Cost Visibility & Allocation

## 1. The problem: Kubernetes destroyed the cost model

Before containers, cost accounting was easy: team A owns VM X; the bill says what X costs. Kubernetes breaks this: **hundreds of pods from many teams share the same nodes, churn constantly, and the cloud bill only shows VMs.** Questions nobody could answer:

- "What does the *checkout service* cost per month?"
- "Which team's namespace is driving this quarter's spend?"
- "How much of what we pay for is actually used?"

Kubecost (founded 2019 by ex-Google engineers, open-source core based on **OpenCost** — now a CNCF project; acquired by IBM/Apptio in 2024) exists to answer these. Its stance in this folder's trio: **measurement first** — you can't optimize what you can't attribute ([cast-ai.md](cast-ai.md) automates, [perfectscale.md](perfectscale.md) recommends; Kubecost's core is *visibility*).

---

## 2. How cost allocation actually works (the interesting part)

Kubecost runs **inside your cluster** and joins two data streams:

```
 (1) Usage: what did each pod consume/reserve, minute by minute?
     └─ from Prometheus (a time series DB! see
        ../02-distributed-storage/time-series-database.md)
        scraping kubelet/cAdvisor metrics
 (2) Price: what does each resource cost?
     └─ from cloud billing APIs (incl. your negotiated discounts,
        RIs/CUDs, spot prices) or on-prem price sheets
```

### The allocation algorithm, step by step

1. **Price out each node** per hour, and split that price into CPU / RAM / GPU components.
2. **Charge each pod** for its share of the node: per resource, charge `max(request, actual usage)` × resource unit price. Using `max` means you pay for what you *reserved* even if idle — reservations block others from scheduling there (this rule is what makes over-requesting visible as cost).
3. **Add shared items**: persistent volumes, load balancers, network egress (network attribution is genuinely hard and approximate).
4. **Distribute cluster overhead**: idle capacity, control-plane fees, and shared services (monitoring, ingress) are spread by configurable policy (proportional to usage, evenly, or weighted).
5. **Aggregate** by any Kubernetes dimension: namespace, deployment, label (`team=payments`), annotation → dashboards, alerts, showback/chargeback reports.

### The three magic numbers

- **Idle cost**: provisioned-but-unrequested capacity — *cluster-level* waste (bin-packing/autoscaler problem).
- **Cost efficiency** = usage / request — *workload-level* waste (over-requesting problem). A pod requesting 4 CPU using 0.2 → 5% efficient.
- **Total waste** ≈ idle + inefficiency. Kubecost's reports splitting these tell you *which tool/fix* you need — that decomposition is the product's core intellectual contribution.

---

## 3. From visibility to action

On top of measurement, Kubecost layers:

- **Rightsizing recommendations**: per-container request suggestions from usage percentiles (e.g., set request = P98 of usage + headroom) — apply manually or via automation.
- **Cluster/node recommendations**: cheaper instance types, consolidation opportunities.
- **Budgets & alerts**: "namespace X exceeded $5k/mo," anomaly detection on spend spikes.
- **Multi-cluster federation**: aggregate many clusters into one view (enterprise tier).

But note: acting is optional/assisted, not autonomous — the philosophical opposite of Cast AI.

---

## 4. FinOps context (one paragraph of business literacy)

This tool category = **FinOps** (cloud financial operations): making engineering teams accountable for the cost of their own workloads. The standard loop is **Inform → Optimize → Operate**; Kubecost is the canonical *Inform* tool for Kubernetes. **Showback** (show teams their costs) vs **chargeback** (actually bill internal teams) is the vocabulary to know. The cultural claim: engineers who *see* their service costs $8k/month write different YAML than engineers who don't.

---

### 5. Distributed-systems angles worth mentioning

1. **It's a metering pipeline**: high-cardinality time series (per-container, per-minute) → aggregation → attribution. All the TSDB machinery — cardinality limits, downsampling, retention — applies directly ([../02-distributed-storage/time-series-database.md](../02-distributed-storage/time-series-database.md)).
2. **Attribution of shared resources is fundamentally arbitrary**: splitting a node's idle cost or a NAT gateway across tenants has no single correct answer — policy, not physics. Good parallel to multi-tenant fairness problems in schedulers.
3. **Measurement shapes behavior** (Goodhart's law): charge on requests → teams cut requests, sometimes below safety; the metric design *is* the incentive design.

---

## 6. Interview questions & strong answers

**Q: How would you compute the monthly cost of one microservice on a shared cluster?**
A: Meter each of its pods' resource reservations/usage over time (Prometheus), price each node-hour from billing data, charge pods `max(request, usage)` per resource, add PV/LB/egress, spread idle & shared overhead by policy, aggregate by the service's labels. Then name the hard parts: network attribution, idle-cost policy, discounts.

**Q: Why `max(request, usage)` and not just usage?**
A: A reservation excludes other pods whether or not you use it — opportunity cost. Charging it also creates the incentive to right-size requests.

**Q: Kubecost vs Cast AI?**
A: Observe vs act. Kubecost = allocation, showback, recommendations (open-core, data stays in-cluster). Cast AI = autonomous rebalancing/spot/rightsizing. Mature orgs often run measurement and automation together.

---

## 7. Key takeaways

1. K8s makes cost attribution hard because **many tenants share churning nodes**; the bill only sees VMs.
2. Core mechanism: **join usage time series with pricing data**, charge `max(request, usage)`, spread idle/shared cost by policy.
3. The **idle vs inefficiency** decomposition tells you whether to fix the autoscaler or the requests.
4. Kubecost = the *Inform* stage of FinOps; open-source core is **OpenCost** (CNCF).
5. Deep insight for interviews: **allocation policy is incentive design** — the meter changes the engineering.
