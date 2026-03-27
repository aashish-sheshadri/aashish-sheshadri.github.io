---
title: "The Curriculum: How You Train an RL System on a Live Cluster (Carefully)"
date: 2026-05-13
draft: true
series: ["HiRL-Scale"]
tags: ["curriculum-learning", "reinforcement-learning", "training", "shadow-mode"]
description: "Six-stage training pipeline, simulation-to-production transfer, and what shadow mode actually revealed."
ShowToc: true
TocOpen: false
weight: 7
---

You cannot train a hierarchical RL system on a live production cluster from random initialization. The first few episodes of random policy exploration on a cluster with 100k pods and real SLO obligations would cause serious problems. Even "safe" random exploration — all actions bounded by hard limits — produces erratic scaling behavior that would degrade enough services to generate a major incident.

So you can't train in production. But the simulation-to-production transfer gap is real: a policy that works perfectly in simulation often fails on real traffic because the simulation doesn't capture the full distributional complexity of production. Shadow mode helps bridge it, but doesn't close it completely.

This post describes my six-stage curriculum, the decisions that shaped it, and the lessons from stages that didn't go as planned.

---

## The Core Curriculum Design Principle

Curriculum learning in RL is the practice of presenting training scenarios in order of increasing difficulty. The theoretical motivation is straightforward: if a policy can't solve easy scenarios, it won't learn anything useful from hard ones. The reward signal will be too sparse and the gradient too noisy.

For HiRL-Scale, "difficulty" has a specific meaning:

```text
Level 1: Single domain, steady-state traffic (no contention, no spikes)
Level 2: Single domain, diurnal patterns and isolated spikes
Level 3: Single domain, correlated spikes across multiple apps
Level 4: Single domain, node pool pressure (forced resource scarcity)
Level 5: Multi-domain, independent scaling (no cross-domain contention)
Level 6: Multi-domain, shared node pool contention
Level 7: Multi-domain, correlated events across domains
Level 8: Adversarial — flash events on lowest-priority domains
```

I don't jump the Soldiers from Level 1 to Level 8. I advance each level only when the policy achieves a defined mastery threshold on the current level (measured on a held-out validation set).

---

![Training Curriculum Stages](/images/07_curriculum_stages.svg)

## Stage 1: Forecaster Pre-training (Offline, ~2 Weeks)

The forecaster is trained first, before any RL training begins. It's a static oracle from the RL system's perspective (the decoupling decision described in Post 4).

Training details: 6 months of per-domain traffic history at 1-minute resolution, with held-out validation periods from each month to capture seasonal variation. The key metric is quantile calibration: does the p80 forecast actually get exceeded ~20% of the time, across domains and time-of-day slices?

At the end of Stage 1, the forecaster backbone is frozen. I keep the output head fine-tunable for the weekly refresh, but the core attention weights are fixed for the duration of RL training.

**What went wrong the first time:** I didn't validate calibration by time-of-day slice, only overall. The aggregate calibration looked good, but the model was systematically overconfident (narrow intervals) during morning ramp-up periods — exactly when traffic is most volatile. The Commander received under-uncertain forecasts during the most uncertain periods. I caught this in shadow mode (Stage 5), retrained with stratified validation, and re-ran the curriculum from Stage 2.

---

## Stage 2: Soldier Pre-training in Simulation (Offline, ~1 Week)

Soldiers train first, without the Commander. This is a deliberate curriculum choice: the Commander's budget allocations are a form of constraint on the Soldiers. If I train both simultaneously from the start, the Commander's random early-training budget allocations are essentially random noise from the Soldiers' perspective. Not useful constraints. Soldiers need to learn the basic scaling task before they can benefit from Commander guidance.

The simulator replays historical traffic traces with injected noise and generates synthetic events:

```text
Simulator components:
  ├── Traffic replayer (historical RPS + Gaussian noise)
  ├── Event injector (spike generator, flash traffic, batch job triggers)
  ├── Resource model (CPU response to replica count, startup latency)
  ├── Node pool model (provisioning delays, capacity limits)
  └── Latency model (queueing theory response to load vs capacity)
```

Training curriculum by phase:

**Phase A (Days 1–2):** Steady-state single-domain traffic. No spikes. The Soldier learns the basic relationship between replica count and CPU utilization, and between CPU utilization and latency. The correct action is obvious and the reward signal is dense. Essentially supervised learning with RL mechanics.

**Phase B (Days 3–4):** Diurnal patterns and single-app spikes. The Soldier learns to anticipate daily traffic cycles using the forecaster input, and learns to scale individual apps in response to isolated load increases.

**Phase C (Day 5):** Correlated multi-app spikes. Multiple apps in the domain spike simultaneously. The Soldier learns to triage — not all apps can be scaled at maximum velocity simultaneously. It must choose where to concentrate resources first.

**Phase D (Days 6–7):** Node pool pressure. The simulator artificially constrains available node capacity, forcing the Soldier to operate within hard resource limits. This is the first time the budget constraint mechanism gets meaningful use — the Soldier can't just scale everything; it has to choose.

Mastery criterion for advancing to Stage 3: Soldier achieves >90% domain SLO adherence on a held-out Phase D scenario set, with average oscillation penalty below a defined threshold.

---

## Stage 3: Commander Pre-training with Frozen Soldiers (Offline, ~1 Week)

The Commander trains with Soldiers frozen at their Stage-2 checkpoint. From the Commander's perspective, the Soldiers are part of the environment: it issues budget directives and observes the resulting domain-level behavior.

This is the critical curriculum stage for the Commander: it needs to learn cross-domain coordination without having to simultaneously cope with Soldiers that are also learning. Frozen Soldiers give the Commander a stable environment.

Commander curriculum by phase:

**Phase A (Days 1–2):** Five domains, each scaling independently with no cross-domain contention. The Commander learns that budget allocations proportional to current load generally keep SLOs happy. Baseline policy.

**Phase B (Days 3–4):** Two domains share a node pool. The Commander must learn that allocating more budget to Domain A implicitly limits resources available to Domain B. Cross-domain resource accounting.

**Phase C (Day 5):** Full five-domain simulation with correlated traffic events. Payment Processing and Merchant Services spike simultaneously (as happens during merchant flash sales and promotional campaigns). The Commander must learn to distribute a scarce budget between two high-priority domains rather than simply maximizing for each independently.

**Phase D (Days 6–7):** Adversarial scenarios. Flash events hit Domain 5 (lowest priority) while Domain 1 is near its SLO threshold. The Commander must learn to deny Domain 5's implicit budget request (by not increasing its allocation) to protect Domain 1. This is the priority inversion scenario from Post 2, solved by a learned policy.

These adversarial scenarios map directly to fintech operational reality. Phase C simulates a payday spike — payment authorizations and merchant API calls surge simultaneously as consumers spend and merchants process. Phase D simulates what happens when a settlement batch window collides with peak checkout traffic — the data pipeline domain demands resources at the worst possible time. The Commander must learn that settlement batches have processing windows with some flexibility in start time; payment authorizations cannot tolerate any delay.

---

## Stage 4: Joint Fine-tuning in Simulation (Offline, ~2 Weeks)

Commander and Soldiers train simultaneously. Both policies unfreeze.

This is where the coupling reward coefficient schedule comes into play. Early in Stage 4, γ (the coupling weight) is low: Soldiers aren't required to follow Commander directives closely, because the Commander's policy is still early in its joint-training evolution. As training progresses and the Commander becomes more reliable, γ increases.

**Key hyperparameter: KL penalty asymmetry.** PPO uses a KL penalty term to prevent the policy from changing too rapidly per update. I use separate KL coefficients for Commander and Soldiers:

- Commander KL coefficient: 3× the Soldier's
- This makes the Commander's policy evolve 3× more slowly per gradient step

Why? The Soldiers need a relatively stable "contract" to adapt to. If the Commander's budget allocations change dramatically every few hundred steps, the Soldiers can't learn reliable behavior within them. A slowly-evolving Commander is a more stable training environment for Soldiers.

**Two-timescale convergence.** The Commander ticks every 60 seconds; Soldiers tick every 10 seconds. In simulation, I run one Commander step for every six Soldier steps, mimicking the production timing relationship. This means the Soldiers see 6 training updates per Commander update — updating faster, consistent with their shorter timescale.

Mastery criterion for advancing to Stage 5: Joint system achieves >85% cluster-wide SLO adherence on a held-out multi-domain adversarial scenario, zero priority inversions (P0 domain degraded while P3 domain healthy), average cascade events below threshold.

---

## Stage 5: Shadow Mode on Production (4 Weeks)

The system runs in parallel with HPA. Every 60 seconds (Commander tick) and every 10 seconds (Soldier ticks), the system computes what actions it would take. It doesn't execute them. The real cluster continues to be managed by HPA. The RL system's actions are logged, and the reward signals that those actions would have generated are computed from real cluster metrics.

Shadow mode serves three purposes:

**Distribution shift detection.** The simulation's traffic model is built from historical data, but the real cluster has patterns the simulation doesn't capture: subtle correlations between domains at specific hours, real incident traffic signatures (retry storms, circuit breaker openings), deployment-triggered metric artifacts. Shadow mode reveals where the simulation and reality diverge.

**Reward hacking validation.** During shadow mode, I run correlation analyses between the reward signal and every metric I can measure — including metrics not in the reward function. If the reward improves but user-facing latency doesn't, something is being gamed. Shadow mode caught several hacking behaviors that simulation missed, because real traffic distribution doesn't contain the "loopholes" the policy exploited in simulation.

**Value function calibration.** The Critic's value function estimate is a prediction of future cumulative reward. In shadow mode, I can compare predicted value to actual reward observed — a direct calibration check. Consistent over-prediction or under-prediction indicates value function generalization failure.

**What I found in shadow mode:** Two significant distribution shift issues. First, the simulation significantly underestimated the frequency of short-duration (1–2 minute) traffic spikes in the Payment Processing domain — caused by merchant flash sale launches and social media-driven checkout surges that the simulation treated as smooth ramp-ups. The trained Commander was too slow to respond to sharp spike onsets. I added a spike-onset feature (rate of RPS change, not just absolute RPS) to the Commander observation and re-ran Stage 4.

Second, the Fraud Detection domain had a strong correlation between payment volume spikes and derived fraud scoring demand that wasn't captured in the simulation (when payment authorization volume surges, fraud scoring requests follow with a 10–30 second lag as transactions enter the scoring pipeline). Adding deployment event flags to the observation fixed this.

I ran shadow mode for four weeks. That duration was a gut call. I don't have a principled stopping criterion. I stopped when the distribution shift analysis stopped surfacing new surprises and the reward correlation looked stable. A more rigorous approach would define a statistical test for simulation-to-reality divergence and exit when it passes. That would also make re-entry decisions (going back to Stage 4) less subjective.

---

## Stage 6: Canary Rollout

One domain at a time. Domain 5 (Internal Platform & Tooling) first — lowest blast radius, highest tolerance for experimentation. The rollout proceeds as follows:

![Canary Rollout Plan](/images/07_canary_rollout.svg)

"RL at X%" means: X% of the domain's scaling decisions come from the RL Soldier; (100-X)% from HPA. Implemented via weighted random routing at the action executor — each scaling decision is either forwarded to the RL action executor or the HPA value, with probability X/100 and (1-X/100) respectively.

**Rollback triggers:** Automatic rollback to HPA if:

- Domain p99 latency exceeds 2× its SLO target for more than 60 continuous seconds after an RL action
- RL-generated actions result in replica counts outside [min_floor, max_ceiling] more than 3 times in 10 minutes
- The reward signal drops below a defined threshold for more than 5 consecutive Commander ticks

After any rollback, I review the event, identify the root cause, make appropriate fixes (observation feature, reward adjustment, constraint tightening), and re-start the canary at 10%.

---

## The Curriculum Decision That Surprised Me Most

I initially planned to train the Commander and Soldiers jointly from the beginning (skip Stages 2 and 3). The argument was that joint training should produce better coordination than sequential pre-training, because the agents co-evolve.

I was wrong. Joint training from random initialization produced a stable failure mode: the Commander quickly converged to always allocating maximum budget to the highest-traffic domain (Domain 1), because this reliably reduced the Commander's worst-domain penalty in early training. The Soldiers for other domains received near-zero budgets and never learned useful scaling policies. The system was stuck in a local optimum — Commander good at allocation under the assumption that only one domain matters, Soldiers for domains 2–5 essentially inoperative.

Sequential pre-training prevented this. Because Soldiers pre-trained before the Commander, all five Soldier policies were viable at the start of joint training. The Commander couldn't exploit Soldier incompetence to its advantage. Joint fine-tuning then improved coordination from a much better starting point.
