---
title: "The Cluster That Learned to Plan Ahead"
date: 2026-04-01
draft: true
series: ["HiRL-Scale"]
tags: ["reinforcement-learning", "kubernetes", "autoscaling", "infrastructure"]
description: "Why reactive autoscaling breaks at scale, and what happens when you teach a cluster to think ahead."
ShowToc: true
TocOpen: false
weight: 1
---

It started with a postmortem I never wanted to write.

A merchant flash sale launched 20 minutes ahead of schedule. Traffic to the payment authorization services doubled in under a minute. Kubernetes HPA did exactly what it was configured to do — it detected the CPU spike and requested scale-out across over 150 checkout-path services simultaneously. Most new pods fit on existing nodes, but dozens of services exhausted their node pool headroom and triggered provisioner requests in a burst. The node provisioner stalled under the queue pressure. New capacity came up four minutes later.

Six minutes is an eternity at scale.

By the time the payment services stabilized, the fraud detection domain — physically sharing node pools with payment processing — had been starved of resources it needed to run inline fraud scoring for a completely unrelated merchant cohort. Two independent SLO breaches. One correlated traffic event. Zero individual components that had misbehaved.

That postmortem was the beginning of HiRL-Scale.

---

## The Scale Problem That Changes Everything

The cluster that motivated this design ran approximately 100,000 pods across about 1,500 services, grouped into five broad domains: Payment Processing & Checkout, Fraud Detection & Risk, Merchant Services & APIs, Data Pipelines & Analytics, and Internal Platform & Tooling. If you've operated at this scale, you already know the uncomfortable truth: **the tools designed for 100 services don't behave the same way at 1,500.**

HPA was designed for a world where services are relatively independent. A service spikes, its CPU goes up, HPA fires, replicas increase. Clean, simple, correct. The problem is that independence is an assumption that fails completely once you have shared node pools, shared quotas, and correlated traffic patterns across thousands of services.

Three failure modes emerge at this scale that simply don't exist at smaller ones:

> In a fintech environment, these aren't abstract availability concerns. A cascade event during peak checkout means failed payment authorizations — immediate, measurable revenue loss. A priority inversion that starves fraud scoring means either transactions process without fraud checks (regulatory exposure) or auth latency spikes while fraud scoring queues (checkout conversion drops). The business cost of cluster-level coordination failures is denominated in dollars and compliance risk, not just SLO breach counts.

**Cascade propagation.** A spike in Domain A consumes node pool capacity. Domain B's pods, which share that pool, can't schedule. Domain B's latency climbs. Domain B fires alerts. Domain B's HPA fires. Now you have two domains competing for the same pool simultaneously.

**The provisioner stampede.** Hundreds of HPAs across two or three domains firing in the same 30-second window isn't hundreds of independent scaling decisions. It's one massive, uncoordinated resource request that creates backpressure on the node provisioner and delays scheduling for pending pods across the cluster. The aggregate behavior is worse than any individual decision.

**Domain priority inversion.** Without any coordination mechanism, there's nothing stopping a low-priority internal service from consuming node capacity that a P0 user-facing service needs. Priority is a concept that exists only in your runbook, not in your autoscaler.

The answer to these problems isn't "tune HPA more carefully." It's to build a system that understands the cluster as a shared resource and coordinates scaling decisions across all 1,500 services simultaneously.

---

## The Insight: Autoscaling Is a Planning Problem, Not a Control Problem

Traditional autoscalers are control systems. They observe current state, compare to a setpoint (target CPU%), and act. This is reactive by design. The controller can only respond to signals that are already present in the system — which means, at minimum, you're always one full metric collection cycle behind reality.

At 100k pods, one collection cycle behind reality is the difference between a graceful scale-out and a cascade.

What I wanted was a system that behaves more like an experienced SRE team: one that looks at traffic patterns, understands what's coming based on historical rhythms, and starts preparing capacity before the spike arrives. One that knows which domains are P0 and protects them even during resource contention. One that treats the cluster as a whole, not as 1,500 independent services.

This is a planning problem. The right tool for planning under uncertainty, with delayed rewards and complex multi-agent interactions, is reinforcement learning.

Specifically, it's **hierarchical reinforcement learning** — and the rest of this series explains exactly how I designed it.

---

## Introducing HiRL-Scale

HiRL-Scale is a two-level hierarchical actor-critic RL system. The design mirrors how experienced operations teams actually handle large-scale incidents.

![Commander-Soldier Hierarchy](/images/01_hierarchy_overview.svg)

**The Commander** operates at the cluster level. Every 60 seconds, it looks at the state of all five domains — aggregate CPU, latency, alert counts, pending pods — and decides how to allocate cluster resources across domains. Critically, it doesn't decide replica counts. It decides _budgets_: how much of the cluster's capacity each domain is entitled to use.

**The Soldiers** (one per domain) operate at the application level. Every 10 seconds, each Soldier looks at the state of its 150–500 applications and decides which ones to scale and by how much — within the budget the Commander just granted it.

Neither level can override the other without consequence. The Commander can't set replica counts directly. The Soldiers can't exceed their budget without paying a penalty in their reward function. The hierarchy creates coordination without central planning becoming a bottleneck.

---

## The Forecaster: Seeing 30 Minutes Ahead

What separates HiRL-Scale from "reactive RL that happens to be faster than HPA" is the **Temporal Fusion Transformer** embedded in the Commander's observation.

Before the Commander decides how to allocate resources, it receives traffic forecasts for each domain: where is RPS headed in 5 minutes? 15 minutes? 30 minutes? The forecasts come with confidence intervals — wide intervals mean uncertainty, and the Commander learns to request more pre-emptive headroom when it's uncertain.

This is the shift from reactive to proactive. When a promotional campaign goes live, the Commander doesn't wait for CPU to spike. It sees traffic arriving in the forecast 10–15 minutes before it lands and starts shifting budget to the payment processing domain in advance. By the time the spike hits, replicas are already up.

The forecaster is a separate model, trained offline on 6 months of historical traffic data and frozen during RL training. This decoupling is deliberate — I'll get into why in Post 4.

---

## What's Coming in This Series

The next posts cover why HPA fails structurally at this scale and how the Commander/Soldier decomposition addresses it. From there I go deep on the forecaster and the two agent designs — that's where most of the interesting failures are. Then the training curriculum, the platform engineering that makes it safe to operate, and the retrospective.

One thing I'll flag now: I don't know yet whether two levels of hierarchy is the right number. There's a reasonable argument for three — a meta-Commander coordinating across clusters, a Commander per cluster, Soldiers per domain. Every time I've sketched that design, it's felt like the right next step. But I haven't built it, and I'd be cautious about anyone who claims hierarchical depth is easy to get right.

A related caveat: many organizations at this scale run multiple smaller clusters rather than one large one. This design assumes a single large cluster. The multi-cluster case is a different problem with different constraints — I'll discuss it later in the series as an open problem.

See you next week.
