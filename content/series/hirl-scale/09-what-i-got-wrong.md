---
title: "What I Got Wrong (And What's Next)"
date: 2026-05-27
draft: true
series: ["HiRL-Scale"]
tags: ["retrospective", "reinforcement-learning", "adoption", "lessons-learned"]
description: "Honest retrospective on what worked, what didn't, and guidance for organizations evaluating RL for infrastructure."
ShowToc: true
TocOpen: false
weight: 9
---

If you've built production ML systems, you know the gap between "the model works" and "the system is reliable" is where most of the time goes. HiRL-Scale is no exception. This post is about that gap: the decisions I'd make differently, and the problems I still don't have good answers for.

---

## Five Things I'd Do Differently

### 1. Start Shadow Mode Earlier — Much Earlier

My original plan treated shadow mode (Stage 5) as a validation step: run it after joint fine-tuning, confirm things look good, proceed to canary. A gate, not a feedback loop.

That was wrong. The distribution shifts I discovered in shadow mode (spike-onset dynamics, deployment-correlated inference traffic) should have fed back into simulation construction. Instead, they required me to partially re-run the curriculum from Stage 4.

If I were starting over, I'd run parallel shadow mode continuously from Stage 2 forward. You don't need to execute shadow actions to collect shadow observations — just run the policy against real observations in log-only mode from the beginning. This gives you a continuously updating calibration of simulation vs reality, so you catch distribution gaps before you're committed to a full re-training cycle.

### 2. Add Spike-Rate Features From Day One

The Commander's observation included absolute traffic level (current RPS) and trajectory (p95 over 15 minutes), but not rate-of-change. I assumed trajectory was a sufficient proxy for velocity.

It isn't. Payment processing spikes triggered by social media-driven checkout surges have a very different RPS-change-per-second profile than spikes triggered by payday cycles or merchant flash sales. The former are shorter-duration and sharper; the latter are longer-duration and more gradual. The Commander couldn't distinguish them from trajectory alone, so it applied the same response to both. Suboptimal for both.

Adding `Δ_RPS / Δ_t` (rate of change of RPS) and its second derivative (acceleration) to the Commander observation made it meaningfully more responsive to sharp-onset spikes. I found this in shadow mode analysis and wish I'd built it in from the start.

### 3. The Coupling Reward Needs an Adaptive Schedule, Not a Fixed One

I implemented the coupling weight γ schedule as a hand-crafted function: start low, increase linearly over the first half of Stage 4, hold at maximum for the second half. This felt principled but was effectively a guess.

In practice, the Commander's reliability didn't track my hand-crafted schedule. There were periods where the Commander was highly reliable early in Stage 4 (for some domains) and periods where it took longer to converge than I expected (for adversarial scenarios).

The right approach is to measure Commander reliability continuously and adapt γ based on observed correlation, not elapsed training steps. Straightforward to implement, would have improved joint training efficiency significantly. It's on the roadmap for the next training cycle.

### 4. Don't Underestimate the Operational Burden of Multi-Domain Priority

I encoded domain priority in the Commander's reward function and expected it to "just work." It mostly did. But the complexity is in the edge cases: What happens when two P0 domains compete? What happens when a P0 domain's alert is a false positive and the Commander protects it aggressively based on noisy signal?

Each of these edge cases required an explicit policy decision: tie-break rules for equal-priority domains, alert quality weighting in the observation, domain-specific false positive rate metadata. None of it was architecturally difficult, but all of it was operationally expensive. Every edge case generated a postmortem, a reward function review, and a re-training cycle.

If I were starting over, I'd spend more time upfront enumerating priority edge cases and encoding explicit rules for them, rather than discovering them through incidents.

### 5. Build the Reward Hacking Detector Before Training, Not After

I built the reward hacking detector (correlation analysis between reward signal and out-of-reward metrics) after I saw the first hacking episode in shadow mode. By then, the policy had already converged to a locally optimal hacking behavior that took additional training to unlearn.

The detector is not expensive to build. It's a statistical correlation analysis that runs weekly. But building it early means you catch hacking episodes during training, before they compound. I lost approximately 3 weeks of training time to hacking behaviors that would have been caught in week 1 if the detector had been running.

---

## Five Things That Worked Better Than Expected

**Sequential curriculum (Soldiers before Commander).** The Stage 2 → Stage 3 ordering prevented the Commander from exploiting Soldier incompetence in early joint training. Avoiding the joint-from-scratch failure mode described in Post 7 was worth the additional 2 weeks of curriculum stages.

**Shared Soldier weights.** Data efficiency, easier debugging (one network to inspect), naturally transferable behavior when domain traffic patterns shift. I had concerns that shared weights would prevent domain-specific specialization; the domain embedding was sufficient to differentiate without separate networks.

**p80 forecast injection, not p50.** The risk calibration decision to use the 80th percentile forecast rather than the median meaningfully reduced SLO breach frequency. The cost — mild over-provisioning during low-uncertainty periods — was smaller than expected because the Commander learned to scale down allocations when the forecast showed declining or stable traffic.

**Strict forecaster/RL separation.** Freezing the forecaster backbone prevented the feedback loop destabilization described in Post 4. Every time I considered relaxing this constraint, I re-read the failure mode analysis and didn't.

**The platform contract.** Formalizing the promise of explainability, override capability, and automatic rollback built enough trust that the system could actually run. Without that contract, a canary rollout stalls at 25% traffic for months, or gets abandoned. Trust is the product, not a nice-to-have.

---

## Open Problems I Don't Have Good Answers For

**Regime changes.** The system is calibrated on 6 months of historical traffic. What happens when the environment fundamentally changes — a new product launch, geographic expansion, a doubling of service count? The simulation goes stale. The forecaster's training distribution diverges from reality. The current approach is to detect regime change via forecaster calibration drift and trigger a re-training cycle. That takes 2–3 weeks. It's the slowest part of the system and the one I'd most like to fix.

**Safe online policy updates.** The forecaster fine-tunes weekly. The RL policy doesn't. It only updates during offline training cycles, which means it can't adapt to production distribution shifts between cycles. Safe online RL is an open research problem — exploration requires trying potentially suboptimal actions, and in production that can mean SLO breaches. I don't have a good answer here yet.

**Multi-cluster coordination.** HiRL-Scale assumes one cluster. A global deployment spans multiple clusters across regions and tiers. Cross-cluster coordination — a meta-Commander balancing load between clusters — is the natural next layer of the hierarchy. The design sketches easily. The training curriculum and safety story for it are where it gets hard. I'd rather admit that than pretend I've figured it out.

---

## Where Things Stand

In simulation and shadow-mode validation (Domains 4 and 5 at full RL control, Domains 1–3 at 50%), results against the baseline HPA system:

| Metric | Change | Business Impact |
| --- | --- | --- |
| SLO breach incidents/week | ~60% reduction | Fewer failed payment authorizations during peak traffic |
| Mean time to scale-out (end-to-end, including pod startup + readiness) | ~55–60% reduction (from ~3.5min to ~1.5min) | Checkout capacity catches up to demand before transactions fail |
| Over-provisioning waste | ~30% reduction (from ~35% idle CPU to ~24%) | Direct compute cost savings across the payment platform |
| Node provisioner churn | ~45% reduction | Reduced API server pressure, more stable scheduling for all domains |
| Domain cascade events/month | ~70% reduction | Fraud scoring no longer starved when payment processing spikes |
| P99 latency during correlated spikes | ~35–40% reduction | Auth latency stays within SLO during merchant flash sales and payday spikes |

These numbers are directional, drawn from simulation and shadow-mode observation windows. Whether they hold at full production traffic across all domains is an open question. The numbers that look good at 50% traffic sometimes look different at 100%, especially for failure modes that only emerge under sustained full load.

The most meaningful number is the cascade event reduction. Those events were the original motivation for this architecture. A 71% reduction means the kind of correlated failure described in Post 1 drops from a weekly occurrence to less than once a month.

---

## Should Your Organization Adopt RL for Infrastructure?

Throughout this series, I've described a system built for a specific context: ~1,500 services across ~100k pods, with domain-specific SLOs tied to payment authorization, fraud detection, and merchant commitments. Not every organization faces this problem. Here's my honest assessment of when hierarchical RL for autoscaling is the right answer, and when it's not.

### When RL Is the Right Answer

You should seriously evaluate RL-based autoscaling if your environment has most of these characteristics:

**500+ services with shared infrastructure.** Below this threshold, the coordination problem isn't severe enough to justify the complexity. HPA with careful tuning, pod priority classes, and possibly KEDA handles 100–300 services adequately. Somewhere between 300 and 500 services, the independent-decision model starts generating coordination failures that tuning can't fix.

**Correlated traffic patterns across domains.** If your services spike independently and rarely interfere with each other, the stampede and cascade problems are infrequent enough to handle reactively. If your traffic is correlated — payday spikes hitting payment and merchant services simultaneously, settlement batches competing with peak checkout, derived traffic patterns where fraud scoring follows payment volume — then coordination becomes essential, not optional.

**Domain-level priority that matters.** If all your services are roughly equal priority, the priority inversion problem doesn't exist. If you have P0 services where degradation means immediate revenue loss or regulatory exposure, and P3 services that should yield during contention, you need a system that continuously enforces those priorities. Not a static configuration that drifts.

**Predictable traffic patterns with known future events.** The proactive advantage of RL with forecasting only materializes if traffic has learnable patterns. Payday cycles, merchant promotion schedules, settlement windows, seasonal shopping patterns — these are all forecastable. If your traffic is genuinely random with no temporal structure, the forecaster adds little value and reactive scaling may be sufficient.

**Existing MLOps capability.** RL for infrastructure is not a first ML project. You need model serving infrastructure, experiment tracking, a feature store or equivalent, monitoring and alerting for model performance, and engineers who understand model lifecycle management. Building all of this from scratch while also building the RL system is a recipe for failure.

### When RL Is Overkill

**Fewer than 200 services.** At this scale, the coordination problem is manageable with simpler tools. KEDA with custom scalers, well-configured HPA with pod disruption budgets, and a competent SRE team with runbooks will handle most scenarios. The engineering investment in RL doesn't pay back.

**KEDA with custom metrics suffices.** If your scaling needs are met by custom metrics (queue depth, request rate, business KPIs) and the primary gap is just "HPA only knows about CPU," then KEDA is the right answer. It's simpler, well-understood, and doesn't require training infrastructure. RL is for when even custom metrics can't solve the coordination and prediction problems.

**No ML operations experience.** If your team has never deployed a model to production, managed model retraining, or operated a feature pipeline, RL for infrastructure is too large a capability jump. Build MLOps maturity first (ideally on a less critical system) before attempting RL on your autoscaling layer.

**Homogeneous workloads.** If all your services have similar traffic patterns, similar resource profiles, and similar priority levels, the hierarchical decomposition doesn't buy much. A single well-tuned autoscaling policy (even a rule-based one) may be sufficient.

### Organizational Readiness Checklist

Before committing to an RL-based autoscaling project, assess whether your organization can answer "yes" to each of these:

1. **Observability pipeline:** Do you have per-service metrics (CPU, latency, RPS, error rate) collected at ≤15-second resolution, with at least 6 months of history retained?
2. **Cluster simulator:** Can you replay historical traffic against a simulated cluster environment? If not, are you prepared to build one? (This took approximately 6 weeks of engineering effort.)
3. **MLOps team:** Do you have at least 2 engineers with production ML experience — model serving, monitoring, retraining pipelines?
4. **Traffic history:** Do you have at least 6 months of per-service traffic data with labeled events (promotions, incidents, deployments, known traffic drivers)?
5. **Model governance:** Does your organization have a model risk management process? In financial services, this is often a regulatory requirement (SR 11-7 or equivalent). In other industries, you'll need to build one.
6. **SLO definitions:** Are your service-level objectives formally defined per domain, with measurable targets and documented business impact of breaches?
7. **Leadership timeline understanding:** Does your sponsoring director understand that this is a 9–12 month project to production, not a quarter? RL systems require simulation, curriculum training, shadow mode, and staged canary rollout. Shortcuts on this timeline create production risk.

If you answered "no" to three or more of these, I'd recommend addressing those gaps before starting an RL project. Each "no" represents a foundation that the RL system will need to stand on. Building it in parallel with the RL work is slower than building it first.

### Team Structure

Based on what I learned building HiRL-Scale, here's the team I'd recommend:

- **1 RL engineer** (Staff/Senior level) — owns reward function design, training curriculum, agent architecture. This person needs genuine RL experience, not just ML. Reward engineering and curriculum design are specialized skills.
- **1 ML engineer** (Senior level) — owns the traffic forecaster, feature pipeline, model serving infrastructure. TFT training, calibration, and serving is a meaningful engineering effort on its own.
- **1–2 platform engineers** (Senior level) — own the telemetry pipeline, action executor, safety constraints, observability, and Kubernetes integration. This is the 80% from Post 8.
- **1 embedded SRE** (Senior level) — owns the human override system, rollback triggers, production monitoring, and acts as the voice of operational reality during design decisions. This person prevents the team from building something that works in simulation but is unoperatable in production.
- **1 sponsoring director** — provides organizational cover, priority alignment, and the patience required for a 9–12 month timeline. Without executive sponsorship, the project will be deprioritized during the first quarter-end crunch.

**Timeline expectation:** 4–5 people × 6 months to a canary deployment on the lowest-priority domain. 9–12 months to full production across all domains. This is aggressive but realistic if the readiness checklist items are already in place. If you're building the simulator, feature pipeline, and MLOps infrastructure simultaneously, add 3–6 months.

### ROI Framing: Translating Technical Metrics to Business Outcomes

The results table above shows technical metrics. For budget conversations with leadership, translate them:

**Over-provisioning waste → compute cost savings.** A 31% reduction in idle CPU at the scale of 100k pods translates directly to infrastructure cost reduction. At typical cloud pricing, even a 10% reduction in over-provisioning at this scale represents significant annual savings. Frame this as the project's self-funding mechanism.

**SLO breach reduction → failed transaction cost.** Each SLO breach during a payment processing spike corresponds to a window where transaction failure rates are elevated. A 62% reduction in breach incidents means fewer payment authorization failures, which means less lost revenue and fewer customer support escalations.

**Cascade event reduction → incident cost.** Cascade events are expensive — they generate multi-domain incidents that require senior SRE attention, produce customer-visible impact, and sometimes trigger contractual SLA credits to merchants. A 71% reduction in cascade events is a 71% reduction in a category of incidents that costs the organization disproportionately in engineering time, customer trust, and contractual exposure.

**Proactive scaling → compliance risk reduction.** In financial services, the ability to demonstrate that scaling decisions are proactive, auditable, and governed is itself valuable. The decision ledger, the model governance process, and the change management for reward functions aren't just engineering best practices — they're evidence of operational maturity that matters during regulatory examinations.

---

## Closing the Series

Nine posts, one system, a lot of failure modes.

Autoscaling at 100k pod scale is a planning problem, not a control problem. Reactive systems — however well-tuned — are structurally mismatched with the coordination and prediction requirements of a cluster at this scale. Hierarchical RL, paired with a proper traffic forecaster and the platform engineering to make it safe to operate, is one path forward.

Whether this specific architecture is the right answer for your cluster, I can't say. The design decisions I made are specific to the traffic patterns, domain structure, and constraints of the infrastructure I was working with. Some were right. Some, as this post made clear, I'd do differently.

What I'm confident in: the architectural instincts are sound. Hierarchical decomposition mirrors how experts actually reason about cluster-wide resource management. Proactive forecasting is necessary at this scale, not optional. And the platform engineering — the telemetry, the safety constraints, the observability, the human override — is not secondary to the ML work. It's the foundation on which the ML work stands.

Thanks for following along.
