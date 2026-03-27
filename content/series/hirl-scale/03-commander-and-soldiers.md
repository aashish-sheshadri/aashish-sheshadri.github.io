---
title: "Commander and Soldiers: Decomposing the Scaling Problem"
date: 2026-04-15
draft: true
series: ["HiRL-Scale"]
tags: ["hierarchical-rl", "reinforcement-learning", "multi-agent", "architecture"]
description: "Why hierarchical RL, why not flat, and how the Commander/Soldier decomposition maps to organizational reality."
ShowToc: true
TocOpen: false
weight: 3
---

The design space for RL-based autoscaling has three obvious options. Two of them don't hold up under scrutiny.

---

## Option 1: One Big Agent

The most natural first thought: replace all 1,500 HPAs with a single RL agent that observes the entire cluster and outputs replica counts for every service.

The observation space is 1,500 services × (however many features per service). At even a modest 20 features per service, that's a 30,000-dimensional input. The action space is 1,500 replica targets. The policy network needs to learn that service 847's replica count should correlate with service 1203's latency — but the signal is buried in an extremely sparse observation where 90% of apps are healthy at any given moment.

This doesn't work for three compounding reasons:

**Sparsity and sample efficiency.** The relevant signals — cross-service interactions, cascade effects — are sparse in the observation and temporally distant from the actions that caused them. Most of the 30,000 dimensions carry no useful signal at any given time. You'd need enormous amounts of simulated experience to train a policy that reliably identifies the interactions that matter from the noise that doesn't.

**Credit assignment.** When the reward signal arrives (domain latency went up), which of the 1,500 simultaneous actions is responsible? The credit assignment problem is the fundamental difficulty of RL; a 1,500-dimensional action space with sparse, delayed feedback makes it extremely difficult to solve reliably.

**Operational opacity.** When the agent makes a bad decision, how do you debug it? "The policy decided to scale service 1203 to 47 replicas based on a 30,000-dimensional input vector" is not a root cause analysis. Platform teams need to understand why the system did what it did.

A monolithic agent fails on sparsity, credit assignment, and interpretability simultaneously.

---

## Option 2: 1,500 Independent Agents

The opposite end of the spectrum: each service gets its own RL agent, observing only its own metrics, outputting only its own replica count.

This immediately reintroduces the coordination problems from Post 2. You've replaced 1,500 HPAs with 1,500 RL agents, but the fundamental issue — that they make independent decisions without awareness of shared resources — is unchanged. You've added training complexity without solving the structural problem.

Independent agents also have a sparse reward problem. A single service's SLO breach is a rare event in steady state. An agent that rarely receives meaningful reward signals converges slowly and unpredictably. This is manageable if you have a few agents; it's operationally unacceptable at 1,500.

---

## Option 3: Hierarchical RL (The One That Works)

When a traffic event hits a cluster with 1,500 services, a senior SRE does not immediately start deciding individual replica counts. They make high-level resource allocation decisions first:

> "Payment processing is P0, protect it at all costs. Fraud scoring is P1, keep it inline with auth. Data pipelines can shed load temporarily. Internal services yield to everyone."

Only after those policy-level decisions are made does the team execute at the service level — and they execute within the constraints those decisions established.

Abstract policy at the top, concrete execution at the bottom. Hierarchical RL formalizes exactly this.

---

## The Commander-Soldier Architecture

![Commander-Soldier Architecture](/images/03_commander_soldier_detail.svg)

**The Commander** operates at 60-second intervals on five-domain aggregates. It knows nothing about individual services — only domain-level CPU, latency, alert counts, and traffic forecasts. Its action space is a budget allocation: what fraction of cluster capacity each domain is entitled to use, plus an urgency signal and a headroom request. Five domains × a few scalars. Tractable.

**The Soldiers** (one per domain) operate at 10-second intervals on their domain's application-level metrics. Each Soldier knows about its 150–500 applications and makes replica decisions within the budget the Commander just granted. The Soldier never considers cross-domain resource contention — that's the Commander's problem, already solved by the time the budget arrives.

This decomposition achieves all three structural requirements from Post 2:

- **Stampede coordination:** The Commander batches all domain-level resource requests into a single, coordinated budget allocation before any Soldier fires. The cluster never sees 400 simultaneous uncoordinated requests.
- **Cascade awareness:** The Commander's observation includes all domains simultaneously. A budget allocation decision for Domain A explicitly accounts for Domains B through E because they're all in the observation.
- **Priority enforcement:** Domain priority is encoded in the Commander's reward function and observation. It's continuously enforced, not configured once and drifted.

---

## Why Actor-Critic for Both Levels

Both the Commander and the Soldiers use an actor-critic architecture. This is worth explaining because it's not the only option.

The **actor** is the policy: given the current observation, what action should I take? The **critic** is the value function: given the current state, what is the expected cumulative reward from here?

The critic is what enables stable learning in complex environments. Without a value function, the policy gradient estimate is extremely noisy — you're using actual episode returns to estimate expected returns, which has high variance. The critic provides a lower-variance baseline. At the timescales I care about (60-second Commander ticks, 10-second Soldier ticks), the environment changes fast enough that high-variance learning is genuinely destabilizing.

For the Commander specifically, the critic's value function serves another role: it encodes the long-horizon consequences of budget allocations. A Commander that allocates too much budget to Domain 1 during a quiet period wastes capacity. The effects of that waste are felt 10–20 minutes later when a spike hits Domain 2 and the budget isn't there. The critic learns to trace these delayed consequences back to the allocation decisions that caused them.

I used PPO (Proximal Policy Optimization) for both levels, with separate KL penalty coefficients. The Commander's KL coefficient is higher, meaning its policy changes more slowly per update. This is deliberate: the Commander needs to be a stable "policy environment" for the Soldiers to adapt to. If the Commander's budget allocations are constantly changing, the Soldiers can't learn reliable behavior within them.

---

## The Coupling Problem: Making Two Levels Cooperate

The most interesting design challenge in hierarchical RL is getting the two levels to cooperate rather than conflict. An adversarial dynamic is easy to produce by accident: the Commander sets a budget, the Soldier ignores it because its local reward for scaling up everything is higher, the Commander raises the budget to compensate, the Soldier still ignores it, and you end up with unconstrained scaling driven entirely by Soldier-level rewards.

The design uses two mechanisms to prevent this:

**Hard constraint:** The Soldier's action space is subject to a simplex projection that enforces the Commander's budget as a hard ceiling. No matter what the Soldier's actor outputs, the projection layer normalizes the replica deltas so their total doesn't exceed the allocated budget. This is differentiable — gradients flow through the projection during training.

**Coupling reward:** The Soldier receives a bonus in its reward function for aligning its actual resource distribution with the Commander's intent. If the Commander signals high urgency for scale-up and the Soldier uses most of its budget on already-healthy applications, it pays a penalty. If the Soldier responds to the urgency signal and concentrates resources where they're most needed, it gets a bonus.

The magnitude of the coupling reward requires careful tuning — too high and Soldiers become sycophants that follow Commander directives even when local evidence says otherwise; too low and Soldiers ignore the Commander entirely. I'll get into the tuning methodology in Post 6.

---

## Shared Weights Across Soldiers

One more non-obvious decision: all five Soldiers share the same network weights.

Five Soldiers, five domains, five separate reward streams. If you train five independent networks, each one sees at most 20% of the experience — and the reward signal for a single domain is sparse because most applications are healthy most of the time. Convergence is slow, and the policies diverge from each other in ways that are hard to predict.

Shared weights solve the data efficiency problem. All five domains contribute to the same gradient update. The Soldiers are differentiated not by their weights but by their observations — each Soldier receives a domain embedding that encodes the traffic character, priority level, and SLO targets of its specific domain. The same underlying policy learns to behave appropriately in each domain by conditioning on that embedding.

This is similar to how multi-task learning works: shared representations, task-specific conditioning. The tradeoff is that you can't have completely different policies per domain. In practice, the domains are similar enough in structure (they're all Kubernetes services) that this tradeoff is fine.

---

## What This Architecture Buys You

Against the three failure modes from Post 2, the decomposition addresses all of them structurally: the Commander batches resource requests rather than letting 400 HPAs fire independently; it observes all domains simultaneously rather than each in isolation; and it enforces priority through a reward function rather than a configuration that drifts.

The next post goes deep on the forecasting layer — the Temporal Fusion Transformer that gives the Commander its forward-looking character — and why the design choices there matter as much as the RL design.

One thing I'm still not sure about: whether shared Soldier weights is the right default. It solved the data efficiency problem cleanly, and the domain embedding was enough to differentiate behavior in practice. But I gave up the ability to have genuinely different policies per domain. For the five domains I worked with — similar enough in structure — that tradeoff felt fine. Whether it holds for domains with more heterogeneous scaling dynamics, I don't know.
