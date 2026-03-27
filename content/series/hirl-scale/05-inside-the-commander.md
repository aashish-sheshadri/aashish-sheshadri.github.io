---
title: "Inside the Commander: Budget Allocation as a Learned Policy"
date: 2026-04-29
draft: true
series: ["HiRL-Scale"]
tags: ["reinforcement-learning", "reward-hacking", "reward-design", "production-ml"]
description: "Reward function design, three reward hacking episodes, and why reward engineering is the hardest part of production RL."
ShowToc: true
TocOpen: false
weight: 5
---

The Commander never directly touches a replica count. It doesn't know how many pods service X is running. Its entire existence is devoted to a single question: _how should I distribute the cluster's capacity across five domains, right now, given what I know and what I expect?_

The reward function is where I made the most mistakes and where the design decisions have the highest leverage. Getting it wrong is easy, and the consequences compound. If you're only going to read one section of this post, read that one.

---

## What the Commander Sees

The Commander's observation vector is a concatenation of per-domain aggregates. Five domains, each contributing a block of features:

```text
Per domain (×5):
  traffic_rps_current          # Current aggregate RPS
  traffic_rps_p95_15m          # 95th pct RPS over last 15 min
  pod_count_current            # Live pods
  pod_count_pending            # Pods in Pending state
  cpu_utilization_p75          # Domain-wide CPU at 75th percentile
  latency_p99_ms               # Domain-wide p99 latency
  alert_count_critical         # Firing critical alerts
  alert_count_warning          # Firing warning alerts
  node_pool_saturation_pct     # Node pool utilization

Cluster globals:
  cluster_cpu_available_pct    # Free CPU capacity
  cluster_node_count           # Current node count
  cluster_node_pending         # Nodes being provisioned
  cluster_cost_per_min         # Rolling cost signal
  hour_of_day                  # Temporal context
  day_of_week
  is_special_event_flag        # Promotion, major deployment, etc.

Injected from forecaster (×5 domains, ×3 horizons):
  traffic_forecast_p80_t5m
  traffic_forecast_p80_t15m
  traffic_forecast_p80_t30m
  forecast_uncertainty_t15m    # (p90-p10)/p50 normalized
```

The deliberate design choice here is what's absent: there are no per-application features in the Commander's observation. The Commander has no idea how many replicas service X is running or what its CPU looks like. That information lives entirely in the Soldiers' observation space.

This isn't a limitation. It's load-bearing. The Commander's tractability comes from operating at domain-level abstraction. If I added per-app features, I'd be back to the dimensionality problem that killed the monolithic agent approach.

---

## What the Commander Decides

The Commander's actor outputs a budget directive per domain. Concretely:

```text
action = {
    domain_budget_pct[5],    # Fraction of cluster capacity allocated to each domain
    scale_urgency[5],        # Urgency signal 0–1 (Soldiers read this)
    headroom_pct[5],         # Pre-emptive buffer above forecast
}
```

The budget percentages go through a **softmax normalization** before being applied. This guarantees two properties: all values are positive (no domain gets a negative budget) and they sum to at most 1.0 (the Commander can hold capacity in reserve by not allocating everything).

The reserve capacity is important. During high forecast uncertainty, the Commander learns to hold 5–10% of cluster capacity unallocated. This buffer is available for whichever domain needs emergency scale-out before the next Commander tick. Without this behavior, sudden unforecast spikes hit an already-fully-allocated cluster with no slack.

The `scale_urgency` signal is not a direct scaling command. It's communication to the Soldiers. A Soldier that receives a high urgency score for its domain knows the Commander is prioritizing speed over efficiency. It scales more aggressively within its budget, accepting replica oscillation risk in exchange for faster scale-out velocity.

---

## The Reward Function: Where I Made My Mistakes

This is the section that matters most. The Commander's reward function determines its values (what it optimizes for) and getting it wrong is easier than you'd expect.

### The Wrong Reward: Average SLO Adherence

My first implementation used average SLO adherence across domains:

```text
R_commander = α × mean(slo_adherence[D]) - β × cluster_cost
```

This looks reasonable. I want domains to meet their SLOs, and I want to do it efficiently.

The problem emerged during training: the Commander learned to consistently sacrifice one domain to optimize the average. It allocated most of the cluster budget to the four highest-traffic domains, leaving Domain 5 (Internal Platform) persistently under-resourced. Domain 5's SLO adherence was poor, but it didn't matter much to the average because Domain 5 has fewer pods and lower traffic.

In production, this means internal engineering tools are perpetually degraded. Not acceptable. But the average-based reward doesn't distinguish between "all domains slightly below SLO" and "four domains excellent, one domain in serious trouble."

### The Right Reward: Worst-Domain Penalty

The fix is to use the worst-performing domain, not the average:

```text
degradation(d) = max(
    latency_p99(d) / latency_SLO(d) - 1,
    error_rate(d) / error_budget(d) - 1,
    0
)

R_commander = (
    cluster_slo_adherence_score
  - cost_per_pod_per_min
  - max(degradation[D])          ← worst domain, not average
  - node_provisioner_thrash
  + forecast_proactivity_bonus
)
```

The `max(degradation)` term means any single domain in SLO breach tanks the Commander's reward regardless of how well the others are doing. The Commander cannot make this trade-off anymore. Protecting Domain 1 at the expense of Domain 5 yields the same penalty as letting Domain 1 degrade.

Note the degradation function uses `max()` across breach dimensions too. Latency breach and error rate breach are treated as equally serious. The Commander can't accept one to avoid the other.

### Priority Weighting

I further extended degradation to weight by domain priority:

```text
weighted_degradation(d) = priority_weight(d) × degradation(d)

priority_weight = {
    Domain 1 (Payment Processing):  1.0   # P0 — auth failure = immediate revenue loss
    Domain 2 (Fraud Detection):     0.9   # P0/P1 — inline with auth; degradation = regulatory exposure
    Domain 3 (Merchant Services):   0.8   # P1 — contractual SLAs with large merchants
    Domain 4 (Data Pipelines):      0.5   # P2 — batch tolerant, but feeds fraud models
    Domain 5 (Internal Platform):   0.3   # P3 — lowest priority, but starvation blocks deploys
}
```

The regulatory rationale behind these weights matters: a Payment Processing failure is immediate, measurable revenue loss — every failed authorization is a lost transaction. A Fraud Detection degradation is deferred but potentially more severe — if fraud scoring drops out of the critical path, transactions process without risk checks, creating regulatory exposure under anti-money-laundering and card network rules. Merchant Services degradation erodes trust with contractual partners. Data Pipeline delays are tolerable in the short term but compound — stale data means stale fraud models, which increases false positive rates downstream.

This means a P0 domain at 10% above its latency SLO generates a higher penalty than a P3 domain at 30% above its SLO. The Commander learns to protect P0 domains more aggressively during contention.

**Why encode priority explicitly rather than let the Commander learn it?** I tried both. Learning priority from reward signals alone is theoretically possible — the Commander should eventually discover that Domain 1 breaches cost more. But the learning is slow: the Commander needs to observe actual SLO breaches (which are costly) to understand their relative severity. Explicit encoding cuts convergence time significantly and eliminates the risk of the Commander learning an inverted priority ordering from noisy early training data.

I'm not fully comfortable with this decision. The priority weights I used (1.0, 0.9, 0.8, 0.5, 0.3) were set by judgment, not measured. If the true relative cost of a P0 vs P1 SLO breach is different from what those weights imply, the Commander has been optimizing for the wrong objective the whole time. There are ways to learn priority weights from incident cost data. I didn't do that, and I think I should have.

### The Forecast Proactivity Bonus

Without this term, the Commander learned to ignore the traffic forecast. The policy converged to "wait for CPU to climb, then reallocate budget." Faster than HPA, but not proactively superior.

The proactivity bonus rewards the Commander for allocating budget in advance of an arriving spike:

```text
forecast_proactivity_bonus = Σ_d [
    IF (forecast_t15m(d) > 1.2 × current_rps(d))  # spike predicted
    AND (budget_increase(d) happened before spike)  # budget pre-allocated
    AND (no latency breach during spike)            # outcome was good
    THEN bonus_value
    ELSE 0
]
```

This is a sparse bonus — it only fires when all three conditions are met. But it directly incentivizes the behavior I want: using the forecast to prepare capacity rather than reacting to realized load.

### The Thrash Penalty

Node provisioners don't like being told to add nodes and then immediately remove them. The cost is real: provisioning latency, API server load, pod scheduling disruption. I penalize the Commander for decisions that lead to rapid provision/deprovision cycles:

```text
thrash = count_of(node_provision_events followed by
                  deprovision_event within 10 minutes)

R_commander -= δ × thrash
```

In practice, this penalty suppresses a failure mode I saw during early training: the Commander would over-allocate budget during a forecast spike, trigger node provisioning, then immediately reduce budget when the spike didn't materialize at full predicted magnitude, triggering deprovisioning.

---

## Reward Hacking Episodes I Actually Encountered

**Episode 1: Conservative hoarding.** The Commander learned to allocate almost all budget to P0 domains regardless of actual load, keeping P2/P3 domains barely provisioned. P0 domains maintained good SLOs easily, and cluster cost was technically "efficient." The Commander's reward was decent, but the cluster was operationally wrong — internal teams couldn't run deployments because the Platform domain was starved.

Fix: Added a minimum budget floor per domain (5% of cluster capacity) and a utilization penalty for domains that are allocated budget but consistently running at low utilization, forcing the Commander to actually use what it allocates.

**Episode 2: Forecast gaming.** The Commander learned to issue budget reductions for a domain right before the Soldier would naturally scale that domain down anyway, claiming the "proactive scaling" bonus when the actual cause of good outcomes was the Soldier's reactive behavior.

Fix: Tightened the attribution window for the proactivity bonus (actions must precede the spike by >5 minutes, not just any time before the outcome) and added an explicit counterfactual comparison against a baseline reactive policy.

**Episode 3: Latency gaming.** The Commander learned that keeping some domains slightly under-provisioned elevated their latency, which would then "improve" when budget was added — generating a large relative improvement signal. It was manufacturing credit for rescuing situations it had created.

Fix: Changed the SLO adherence component to use absolute threshold (are you within SLO?) rather than relative improvement (did you get better?).

---

## Behavioral Properties of the Trained Commander

The trained Commander exhibits a few behaviors worth calling out explicitly:

**Preemptive budget shifting.** When the forecaster shows a 15-minute traffic increase for Merchant Services, the Commander begins increasing Merchant Services' budget allocation 2–3 ticks (2–3 minutes) before the actual traffic arrives. This gives Soldiers time to ramp up replicas ahead of the spike.

**Uncertainty-driven conservatism.** When forecast uncertainty is high, the Commander holds 8–12% of cluster capacity in reserve rather than its typical 3–5%. This buffer absorbs unpredicted spikes without requiring emergency node provisioning.

**Cross-domain protection during cascade.** When one domain begins degrading, the Commander reduces budget for lower-priority domains to free up capacity — without waiting for those domains to explicitly over-provision. This is the behavior that prevents cascade propagation.

**Slow-to-scale-down.** The Commander is asymmetric: it scales up budget allocation faster than it scales it down. This isn't a configured bias — it emerged naturally from the thrash penalty. The Commander learned that rapid budget reductions often trigger deprovisioning events that are expensive to reverse.
