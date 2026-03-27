---
title: "The Forecaster: Teaching the Cluster to See 30 Minutes Ahead"
date: 2026-04-22
draft: true
series: ["HiRL-Scale"]
tags: ["time-series", "tft", "forecasting", "ml-systems"]
description: "Temporal Fusion Transformer design, frozen forecaster rationale, and what happens when your traffic predictions are wrong."
ShowToc: true
TocOpen: false
weight: 4
---

The Commander's proactive character — the thing that separates HiRL-Scale from "reactive RL that's faster than HPA" — comes from a single component: the traffic forecaster.

I use a Temporal Fusion Transformer (TFT) to produce per-domain RPS forecasts at 5, 15, and 30-minute horizons. I feed the 80th-percentile forecast (not the median) into the Commander's observation. I train the forecaster offline and freeze it during RL training. Each of these choices had consequences I didn't fully anticipate before making them.

---

## Why Traffic Forecasting Changes the Problem

An autoscaler that can only see current state is always reacting. Even if it reacts instantaneously — even if it requests new pods the moment CPU starts climbing — it's already behind. Pod startup time, node provisioning time, and warmup time mean that capacity requested today is available in 2–6 minutes. If the traffic spike lasts 3 minutes, reactive autoscaling missed the window entirely.

A forecaster changes this. If the Commander knows that Domain 3's traffic is going to increase 40% over the next 15 minutes, it can start shifting budget to Domain 3 now. Soldiers can start ratcheting up replica counts before latency climbs. By the time the actual traffic arrives, the cluster is already provisioned for it.

More importantly: if the Commander knows the spike is coming _before_ it hits, it can coordinate a graceful node provisioning request — one batched request to the provisioner rather than a cascade of emergency requests triggered by simultaneous CPU alarms.

The forecaster transforms autoscaling from a reaction problem into a planning problem.

---

## The Forecasting Challenge at This Scale

Traffic at 100k pod scale has properties that make off-the-shelf forecasting approaches brittle:

**Multi-scale seasonality.** Traffic has hourly patterns (office hours), daily patterns (weekday vs weekend), and weekly patterns (some domains spike on Mondays, others on Fridays). A model that captures only one scale will produce systematically biased forecasts for the others.

**Domain heterogeneity.** Five domains with fundamentally different traffic characters. Payment Processing is event-driven — it spikes sharply during merchant flash sales, payday periods, and promotional campaigns. Fraud Detection is workload-driven — its traffic is derived from payment volume and follows a smoother curve tied to transaction patterns. Data Pipelines is batch-driven — it spikes during settlement batch windows and nightly reconciliation runs. A shared model needs to capture these different dynamics without letting one domain's patterns pollute another's.

**Known future covariates.** Some events that cause traffic spikes are known in advance: scheduled promotions, planned deployments, marketing campaigns. A good forecaster should be able to use this information. Most time-series models treat all future inputs as unknown.

**Probabilistic output.** The Commander needs more than a point estimate. It needs to know whether the forecast is highly confident or highly uncertain. The appropriate response to a low-confidence forecast is to request more headroom. A deterministic model can't provide this.

---

## Why the Temporal Fusion Transformer

I evaluated several candidates before landing on TFT:

**LSTM / GRU:** Simple to train, but struggles with multi-scale seasonality without explicit seasonal decomposition. Doesn't handle known future covariates natively. Requires separate calibration for uncertainty. I'd be engineering around its limitations rather than using it.

**N-BEATS:** Strong univariate point forecaster (top-ranked on M4). Doesn't handle exogenous inputs — no way to tell it about a scheduled promotion. Deterministic.

**Prophet:** Designed for business time series with strong seasonality and known events. Too rigid — the functional form is fixed, and it's not designed for multi-variate forecasting with domain interactions.

**Temporal Fusion Transformer:** Handles multi-scale seasonality via variable-length lookback windows. Native support for static covariates (domain identity), time-varying known future covariates (calendar events, promotion flags), and time-varying unknown past covariates (historical RPS). Produces quantile forecasts natively. Attention weights are interpretable — you can see which time steps the model focused on.

TFT won on capabilities, though it's more complex to train and serve than the alternatives.

---

## Architecture Details

![TFT Forecaster Architecture](/images/04_tft_forecaster.svg)

**Variable Selection Networks** are one of TFT's most useful features here. They learn, per domain, which input channels actually matter. For the Payment Processing domain, the variable selection network learns to weight `is_payday_flag` and `is_promotion_flag` heavily — payday cycles and merchant promotions are the dominant drivers of payment traffic spikes. For Fraud Detection, it learns that payment volume forecast matters much more than any external event flag — fraud traffic is derived, not independent. This per-domain selectivity happens automatically, without feature engineering.

**The quantile output head** is what gives probabilistic forecasts. Rather than predicting a single value at each horizon, the model predicts multiple quantiles simultaneously, trained with the pinball loss function. The result: a p80 forecast calibrated to be exceeded only 20% of the time, across the entire validation distribution.

---

## What Gets Injected Into the Commander's Observation

Not all the model's outputs go to the Commander. I inject:

```text
# Per domain, three horizons:
traffic_forecast_p80_t5m[D]    # 5-min ahead, 80th percentile
traffic_forecast_p80_t15m[D]   # 15-min ahead, 80th percentile
traffic_forecast_p80_t30m[D]   # 30-min ahead, 80th percentile

# Uncertainty signal:
forecast_uncertainty[D]        # (p90 - p10) / p50 at t+15m
                               # normalized ratio, higher = less certain
```

I use the p80 forecast, not the p50 (median). This is a deliberate risk calibration: I'd rather pre-provision slightly more capacity than needed than be caught under-provisioned when a spike materializes. The cost of mild over-provisioning is a few extra pods for a few extra minutes. The cost of under-provisioning at this scale is SLO breaches across hundreds of services.

The uncertainty signal (`forecast_uncertainty`) is equally important. A Commander that ignores forecast uncertainty will treat a high-confidence forecast and a wild guess the same way. The uncertainty signal drives the Commander to request more headroom buffer when the forecast is uncertain — and the Commander's reward function includes a proactivity bonus that incentivizes it to learn to read that signal correctly.

---

## The Feedback Loop Trap: Why I Froze the Forecaster

Early in development, I tried training the forecaster jointly with the RL agents. The logic seemed reasonable: as the RL agents learn better scaling policies, the traffic patterns they observe should change (better scaling → rarer latency events → fewer retry storms → lower observed RPS). Shouldn't the forecaster learn from this?

No. Here's the failure mode:

1. Commander scales Domain A pre-emptively based on forecast
2. Pre-emptive scaling reduces latency, which reduces client retry storms
3. Traffic RPS is now lower than it would have been without the scaling
4. Forecaster observes lower-than-forecast RPS, updates to predict lower traffic
5. Commander receives lower forecast, scales less aggressively
6. Domain A gets under-provisioned, latency climbs, retry storms return
7. Forecaster now observes higher RPS again

This is a classic feedback loop. The forecaster's training distribution shifts based on the Commander's actions, which changes the Commander's actions, which shifts the distribution again. The joint system oscillates rather than converging.

The fix is strict separation: the forecaster is trained offline on historical data, frozen, and treated as a static oracle by the RL training loop. I refresh it weekly with a frozen-backbone fine-tune to handle traffic distribution drift, but this fine-tuning is independent of the RL training.

I'm not fully satisfied with this answer. The feedback loop problem is real, but freezing the forecaster means it can never improve from what the RL system actually does — which feels like leaving information on the table. There's probably a way to do safe joint training with the right lag structure or causal separation. I haven't found it yet, and I wasn't willing to risk destabilization in production to find out.

---

## Calibration: The Metric That Actually Matters

Most time-series forecasting papers optimize and report MAPE or RMSE. These are fine for point forecasting. For this use case, they're the wrong metrics.

I care about **quantile calibration**: does the p80 forecast actually get exceeded 20% of the time? If the p80 forecast is exceeded 40% of the time, the Commander is systematically making under-provisioned allocations. If it's exceeded only 5% of the time, I'm over-provisioning unnecessarily.

After initial training, I validated calibration domain by domain, time-of-day slice by time-of-day slice. There was significant calibration drift across time slices: the model was well-calibrated for steady-state weekday traffic but under-confident (too wide intervals) for weekend patterns and over-confident (too narrow intervals) for event-driven spikes.

I fixed this with post-hoc recalibration using conformal prediction — a model-agnostic technique that guarantees coverage properties on held-out data. The recalibration is applied per domain, per time-of-day slice, ensuring that the Commander receives properly calibrated uncertainty estimates regardless of when it's operating.

---

## Serving Infrastructure

The forecaster runs as a sidecar service alongside the Commander inference service. On each 60-second Commander tick, it pulls 24h of per-domain history, assembles known future covariates (calendar events, deployment flags, time context), and produces quantile forecasts across all five domains. The forecasts are cached and injected into the Commander's observation before inference runs.

If the forecaster service is unavailable, the Commander falls back to exponential smoothing inline. Less accurate, but always available. An RL agent that silently degrades to bad inputs is worse than one that explicitly falls back to a simpler model.

---

If the forecaster is the system's eyes, the reward function is its values. Getting the values right turned out to be the hardest part of the whole project.
