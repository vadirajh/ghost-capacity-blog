---
layout: post
title: "Stop Chasing Ghost Capacity: Turning “Free-on-Paper” Into Real Free at Scale"
date: 2025-08-27
---

> **Results:** GC runtime ↓ **20%**, storage reclamation ↑ **30%** in our largest region.  
> **How:** added the right metrics, ranked high-yield nodes, used **concurrency-based per-node tokens**, re-scored every **60s**.

<!-- Paste the rest of your article here -->

.
Why uniform GC broke at large scale
Uniform scheduling assumes servers are “roughly equal.” That held in smaller regions. In our biggest region — with thousands of servers — three scale effects slowed capacity return:

Heavy-tailed variance. A minority of servers held a disproportionate share of gc-able extents. Uniform GC wasted cycles on low-yield nodes while high-yield nodes waited.
Straggler/fan-out penalties. Scattering GC everywhere created many small, low-yield cleanups; time-to-reclaim was gated by slow/busy nodes.
Hot-node blindness (no metrics). We initially had no per-server GC/IO signals. Uniform GC sometimes piled extra work onto already busy nodes, inflating high-percentile latencies.
Net effect: capacity looked “free” in metadata but took too long to become allocatable headroom.

Step 1 — Instrument first (add the missing metrics)
Before changing scheduling, we shipped lightweight per-server metrics:

GC-eligible bytes/extents (our yield signal)
I/O pressure (queue depth, disk busy%, p95/p99 latency)
Recent GC work (completions per node) for anti-bursting
Why fragmentation slows reclaim
GC frees space at segment granularity. If a segment is u% invalid, you must relocate (1−u)% live data to free it. Cleaning yield:

yield ≈ u / (1 — u)​

u = 20% → yield 0.25 → move 4B to free 1B (bad).
u= 60% → yield 1.5 → free 1.5B per 1B moved (good).
Fragmentation keeps u low across many segments → low yield, high write-amplification, slow capacity return.

Step 2 — Prioritize high-yield nodes
Measure → Rank → Schedule → Refresh

Rank each server by a composite score (recomputed every 60s):
score(node) = w1*gc_extents – w2*io_pressure – w3*recent_gc + w4*age

Concurrency tokens (per node). Each node gets a small integer budget — the max number of GC jobs that may run concurrently on that node.
Start with base_concurrency = 1–2.
Make it adaptive to I/O pressure:
scale_i = clamp(1 - α * normalize(io_pressure_i), 0, 1) max_conc_eff_i = max(floor(base_concurrency_i * scale_i), conc_floor)
For example : A cool node keeps 2 tokens; a hot node drops to 1 or 0.

Fairness. From the top-K nodes by score, assign work via weighted round-robin (weight = score), spreading across racks/failure domains.
Cooldown & aging. After a job completes on a node, apply a small cooldown penalty to its score so we don’t re-hammer it immediately; use aging so low-score nodes eventually get service.
Pseudocode (60s re-score + concurrency tokens)
every 60s:
  for node in nodes:
    score[node] = w1*gc_extents[node]
                - w2*io_pressure[node]
                - w3*recent_gc[node]
                + w4*age[node]
    max_conc_eff[node] = max(floor(base_conc[node] * scale(io_pressure[node])), conc_floor)
  heap = max_heap(score)

every tick (e.g., 200ms):
  for node in weighted_rr(topK(heap, K), weight=score):
    if active_gc[node] < max_conc_eff[node] and healthy(node):
      assign_gc(node)           # async start, consumes 1 token
      active_gc[node] += 1

on_gc_finish(node):
  active_gc[node] -= 1
  recent_gc[node] = decay(recent_gc[node]) + 1
Details on how we identify the right candidate are covered here.

Worker guardrails: Even with Concurrency control, each GC worker processes in bounded chunks and yields briefly between chunks if local p99/queue depth blip. If health gates trip, the worker aborts early and the job is requeued elsewhere.

Results
In our largest region:

GC runtime ↓ 20% (wall-clock per cycle)
Storage reclamation rate ↑ 30% (allocatable GiB/hour)
Side effects: steadier high percentiles during churn and fewer GC-vs-foreground collisions
Rollout plan
Instrument first (compute scores; no scheduling change).
Turn on concurrency tokens; verify hot-node protection.
Enable top-K targeting at low weight; watch capacity surface rate and p95/p99.
Increase weights/K gradually; keep a feature flag to fall back to uniform.
What to watch on dashboards
active_gc per node, queue length, and max_conc_eff over time
Capacity surfaced (GiB/hour) and time-to-reclaim (e.g., 1 TiB)
p95/p99 of writes or metadata ops during churn
Skip reasons (no tokens, health gate, cooldown) to validate fairness vs. protection
When uniform GC is still fine
Small regions with low variance across servers
Low-churn windows or maintenance batches
Fallback mode (“clean a little everywhere”)
At large scale, targeted GC becomes the default; uniform remains a simple Plan B.

Future enhancements
Job sizing & weighting. Tag GC tasks (light/medium/heavy) so heavy ones consume 2 tokens or are chunked smaller.
EC-aware cost. Prefer segments whose reclaim requires fewer parity recomputes/hops; penalize partial-stripe cleans.
Predictive scoring. Add short-term forecasts (Exponentially Weighted Moving Average aka EWMA trends, bandit/explore-exploit) to chase emerging high-yield nodes sooner.
PID-style auto-tuner. Adjust base_concurrency per node to hold a target p99 while maximizing GiB/hour.
Rack-aware fairness. Ensure at least one token per rack (when healthy) to keep surface rate resilient to single-rack hotspots.
Rescore at 5s-30s intervals . A lot can change in 60s a more frequent rescore will give a better view of the current world and hence a better map.
Takeaway
Uniform GC scales until variance dominates. With the right metrics, concurrency-based per-node tokens, and a 60-second re-score, prioritizing high-yield nodes turned ghost capacity into real headroom — cutting GC runtime by 20% and boosting reclamation by 30% in our biggest region.
