---
title: "Inside the Soldiers: Scaling 400 Apps Without Breaking the Budget"
date: 2026-05-06
draft: true
series: ["HiRL-Scale"]
tags: ["multi-agent", "reinforcement-learning", "attention-mechanism", "distributed-systems"]
description: "Attention-based observation spaces, simplex projection for budget compliance, and multi-agent coordination at scale."
ShowToc: true
TocOpen: false
weight: 6
---

Each Soldier has one constraint: stay within the budget the Commander allocated. Within that budget, it has a hard job — decide which of its 150–500 applications to scale, by how much, and how fast, every 10 seconds.

---

## The Observation Challenge: 400 Apps Is Too Many to Watch Naively

The Soldier for a 500-app domain can't have a raw 500-app × N-feature input vector. That's not a theoretical concern — it's a practical one. At 20 features per app, that's a 10,000-dimensional input where the vast majority of features are irrelevant at any given moment — 90% of apps are healthy 90% of the time. The observation is extremely sparse, and policy networks struggle to extract the handful of meaningful signals from that much noise without prohibitively large data requirements.

Two complementary mechanisms solve this:

### Mechanism 1: Attention Over App Embeddings

Each application has a learned **embedding vector** — a dense representation of its "traffic identity." These embeddings are pre-trained offline using a contrastive learning approach: apps with similar traffic shapes across their history are pulled close together in embedding space; apps with different patterns are pushed apart.

At inference time, the Soldier runs cross-attention between a domain query vector (built from domain aggregates) and all app embeddings simultaneously. The attention weights determine which apps matter to the current observation:

```text
domain_query = MLP(domain_aggregates)    # compact domain state
app_keys     = W_k × app_embeddings      # all N app embeddings projected
attention    = softmax(domain_query × app_keys^T / √d)

# Apps near SLO breach → high attention weight
# Healthy apps → low attention weight
```

The SLO proximity signal is baked into each app's current embedding by concatenating its static historical embedding with a small real-time feature vector (current CPU%, current latency as fraction of SLO, current RPS vs historical mean). This means apps that are currently close to breach receive higher attention weights, even if they historically look similar to healthy apps.

The output is a fixed-size aggregated domain representation regardless of N. Whether the domain has 200 or 500 apps, the Soldier's input dimension doesn't change.

### Mechanism 2: Explicit Top-K Focus

Attention over embeddings gives the Soldier a smooth compressed view of the domain. But it can lose precision on the exact metrics of the most critical apps — the ones that most urgently need scaling decisions.

I supplement the attention mechanism by explicitly extracting the **top-20 most at-risk applications** and concatenating their raw features directly:

```text
at_risk_score(app) = max(
    cpu_utilization / cpu_slo,
    latency_p99 / latency_slo,
    pending_replicas / current_replicas
)

top_20 = argsort(at_risk_score, descending)[:20]

soldier_observation = concat(
    domain_aggregates,
    commander_directive,
    attention_output,           # compressed N-app view
    top_20_raw_features,        # precise view of 20 most critical apps
)
```

Together: the attention gives breadth, the top-K gives precision where it matters most.

The top-K cutoff (K=20) is a heuristic. I tested a range and 20 was stable, but I don't have a principled argument for why it's better than 15 or 30. It's possible that in domains with highly heterogeneous app sizes, the right K is different — and that the fixed cutoff is quietly hurting performance in cases where more than 20 apps are simultaneously at risk during large correlated spikes.

---

## Action Space: Replica Deltas, Not Absolute Targets

The Soldier outputs a replica delta for each application in its domain — a signed integer representing how many replicas to add or remove relative to the current count.

```text
A_soldier[app_i] = replica_delta[i]   # can be negative (scale down)
                 + scale_velocity[i]  # 0–1, how fast to ramp
```

Why deltas rather than absolute target replica counts?

**Training stability.** A service running 500 replicas and a service running 5 replicas require the actor to output values that differ by 100×. This creates gradient scaling problems. Deltas are naturally bounded — the typical delta is a small fraction of the current count — giving uniform gradient magnitudes across all apps.

**Action magnitude independence.** A "+10 replicas" action means something very different for a 10-replica service (100% increase) versus a 1000-replica service (1% increase). Deltas don't fully solve this — I also normalize deltas by current replica count as a percentage change — but the conceptual representation as "change" rather than "target" aligns better with how operators think about scaling events.

**Safety.** Minimum and maximum replica bounds are easy to enforce on deltas: clip to `[min_replicas - current, max_replicas - current]` before applying. Enforcing bounds on absolute targets requires knowing the current count at action time, which introduces a race condition if the count changes between observation and action.

---

## The Budget Constraint: Simplex Projection

The Commander allocates a budget to each Soldier. The Soldier cannot exceed it. This isn't a soft preference — it's a hard architectural constraint.

I enforce it via **simplex projection**: after the actor network generates unconstrained replica deltas, I project the positive deltas (scale-ups only) onto the simplex defined by the Commander's budget.

```text
# Raw actor outputs
raw_deltas = actor_network(observation)

# Only positive deltas count against budget
scale_ups = max(raw_deltas, 0)

# Project onto budget simplex:
# Σ(scale_ups × pod_resource_cost) ≤ budget
projected_scale_ups = simplex_project(scale_ups, budget_capacity)

# Final action: projected ups + raw downs (scale-downs are always allowed)
final_deltas = projected_scale_ups + min(raw_deltas, 0)
```

Critically, this projection is **differentiable**: gradients flow through it during backpropagation, so the actor learns to produce feasible solutions rather than relying on the projection to rescue infeasible ones.

In practice, after a few hundred training episodes, the actor rarely produces projections with large corrections — it learns to allocate naturally within budget. The projection becomes a safety net rather than a load-bearing mechanism.

Scale-downs are unconstrained by the budget. The Soldier can always reduce replicas (subject to the minimum floor). Resource recovery helps the Commander, so there's no budget reason to prevent it.

---

## The Coupling Reward: Aligning with the Commander

The Commander sends each Soldier a directive: budget allocation, urgency score, headroom target. The Soldier receives this as part of its observation. But observations don't automatically create incentives — a Soldier could receive a high urgency signal and still spend its budget on already-healthy apps if the local reward for doing so is higher.

This is where the coupling reward comes in:

```text
alignment = cosine_similarity(
    actual_resource_vector,        # how budget was actually allocated across apps
    commander_intended_vector      # what the urgency/headroom signal implies
)

R_soldier += γ × alignment
```

The `commander_intended_vector` is derived from the Commander's urgency signal: high urgency means "concentrate resources on at-risk apps"; low urgency means "balanced allocation is fine." If the Soldier concentrates resources consistent with urgency, alignment is high and it receives a bonus.

### Tuning the Coupling Coefficient γ

This was one of the trickier hyperparameters. The consequences of getting it wrong in either direction:

**γ too high — Soldiers become sycophants.** The Soldier follows Commander directives even when its local observation clearly shows a different priority. A spike is happening in App 47 right now, but the Commander's 60-second-old directive says to concentrate on App 12. The Soldier follows the directive anyway, App 47 breaches SLO, and the Soldier's own domain SLO reward tanks. Worse outcomes than if the Soldiers had just acted independently.

**γ too low — Soldiers ignore the Commander.** The coupling signal is too weak to overcome the local incentive to scale everything up. Soldiers act as independent optimizers. You're back to the HPA stampede problem.

I set γ with a schedule: it starts low during early joint training and increases as the Commander becomes more reliable (measured by the correlation between Commander directives and subsequent optimal Soldier actions). A more reliable Commander warrants stronger coupling. This schedule is evaluated on a held-out validation set during training.

---

## The Soldier's Full Reward Function

```text
R_soldier = (
    α × domain_slo_adherence_score    # Primary: are domain SLOs met?
  - β × budget_excess_penalty         # Don't exceed Commander allocation
  + γ × commander_alignment_bonus     # Follow Commander intent
  - δ × oscillation_penalty           # Don't thrash replicas up-down
  + ε × resource_efficiency_score     # Don't waste allocated budget
)
```

### Oscillation Penalty

Replica oscillation — scaling up, then down, then up again within a short window — is expensive. Each scale event has a cost: pods take time to start, traffic shedding during pod termination, scheduler churn. The oscillation penalty fires whenever a scale-down follows a scale-up for the same app within a 2-minute window:

```text
oscillation_penalty += count_of(
    scale_up(app, t) followed by scale_down(app, t + Δ)
    where Δ < 120 seconds
)
```

This shaped the Soldier toward a "scale up confidently, hold, then scale down carefully" pattern — which is exactly what experienced operators do manually.

### Resource Efficiency Score

A Soldier that requests 50 extra replicas and only uses 5 of them is wasting budget that other apps in the domain could have used. The efficiency score penalizes systematic over-allocation:

```text
efficiency(app) = actual_load / allocated_capacity
efficiency_score = mean(efficiency[apps in domain])
```

Apps that are chronically under-loaded relative to their replica count drag down the efficiency score. This drives the Soldier toward right-sizing rather than defensive over-provisioning.

---

## Behavioral Properties of the Trained Soldiers

**Triage ordering.** Soldiers learn to prioritize scale-up for apps with the highest SLO breach risk, not the highest absolute load. An app at 95% of its latency SLO gets attention before an app at 70% CPU (which may have headroom). This matches how experienced operators think about priority during a spike.

**Budget-aware scale-down.** When Commander budget is reduced (the Commander is redistributing to a higher-priority domain), Soldiers learn to scale down their healthiest apps first — preserving replicas for apps that actually need them. This cooperative behavior emerges from the alignment bonus; I didn't explicitly encode it.

**Scale velocity response.** High urgency from the Commander → Soldiers ramp up faster, accepting some oscillation risk. Low urgency → Soldiers ramp up gradually. The scale velocity output from the actor maps directly to ramp rate in the action executor.

**Shared weights, differentiated behavior.** Despite sharing network weights, Soldiers in different domains behave differently because their observations differ. The Payment Processing Soldier has learned payday and merchant flash sale signatures — it scales more aggressively when it detects the `is_payday_flag` or rapid RPS acceleration characteristic of promotional campaigns. The Fraud Detection Soldier has learned that its traffic is derived from payment volume — when the payment forecast rises, it proactively scales even before its own RPS increases, because it knows fraud scoring demand follows authorization demand with a short lag. The domain embedding in the observation is sufficient to differentiate behavior without separate networks.
