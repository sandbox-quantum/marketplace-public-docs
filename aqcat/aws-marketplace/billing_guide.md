# AQCat — Billing Guide

> Note that this guide is for using the real-time inference endpoint, which is metered per MLP inference call (vs. batch transform, which is metered by instance hour but is generally not recommended).

How you are charged for the **AQCat** model package, for both modes, and how to estimate and
control cost. There are **two cost components**:

1. **AWS Marketplace software charge** — metered per successful inference (the `ConsumedUnits`
   below), via the `X-Amzn-Inference-Metering` header SageMaker reads from each response.
2. **SageMaker infrastructure** — the per-second instance cost while a real-time endpoint is running
   (or a batch job is active), independent of the software charge.

> **Failed requests (4xx/5xx) are not billed** for the software charge. You still pay infrastructure
> for as long as the endpoint is up — so **delete the endpoint when you're done** (see each
> walkthrough's clean-up step).

The metering header looks like:

```
X-Amzn-Inference-Metering: {"Dimension":"inference.count","ConsumedUnits":N}
```

`N` is computed differently per mode.

---

## MLP mode (`mode: "mlp"`)

```
ConsumedUnits = number of instances in the request
```

One forward pass per instance — so a request with 50 structures in `instances` bills 50 units.
Simple and linear in the number of structures you submit.

---

## Adsorption workflow (`mode: "min-adsorption-energy-workflow"`)

The workflow does **not** bill a flat placement count. It bills the number of **model forward
passes (MLP calls)** the run actually makes — **one per optimizer (LBFGS) step**, summed over every
relaxation:

```
relaxations  = bulks × facets × adsorbates × num_placements      (per instance, summed over instances)
ConsumedUnits = Σ optimizer steps over all relaxations
              ≤ relaxations × max_steps
```

Two things drive the count:

- **How many relaxations** — every `(bulk, facet, adsorbate)` combination is relaxed
  `num_placements` times. `num_placements` is **per combination**, so adding facets or adsorbates
  multiplies the work.
- **How long each relaxation runs** — each takes **up to `max_steps`** optimizer steps, but stops
  early once it converges to `fmax`. So `relaxations × max_steps` is the **upper bound**; real cost
  is usually lower.

### Worked example

Request: 1 bulk, facets `[[1,1,1],[1,0,0]]` (2), adsorbates `["*CO","*OH"]` (2),
`num_placements = 3`, `max_steps = 150`.

```
relaxations  = 1 × 2 × 2 × 3            = 12
ConsumedUnits ≤ 12 × 150                = 1800   (upper bound)
```

If relaxations converge in ~60 steps on average, the actual bill is ≈ `12 × 60 = 720` units.

---

## Controlling cost

- **Fewer combinations per request** — each extra facet or adsorbate multiplies the relaxation
  count. Screen broadly first, then drill in.
- **Lower `num_placements`** — fewer placements per combination (default 5, min 1).
- **Lower `max_steps`** — caps how long each relaxation can run (default 150, max 200). Lower caps
  reduce the worst case but may leave some placements unconverged (`is_relaxed: false`).
- **`converged_placements_only` / `filter_anomalous_placements`** — affect what's returned, not what's
  metered (you're billed for the compute already done).
- **Delete the endpoint** after real-time use, and **delete the model** after any job — infrastructure
  bills for as long as the endpoint is up.

---

## Delivery path & cost

Use the **real-time endpoint** and, for many structures/combinations, **mini-batch** them by putting
multiple entries in the `instances` list of one request — that is the recommended high-throughput
path and it does not change how `ConsumedUnits` is computed (it's still per instance / per forward
pass). The **batch-transform endpoint is not recommended** (see each workflow's README); it carries
the same software metering but adds S3 plumbing for little benefit in AQCat's iterative use cases.

> Authoritative source: `take_metering_units` and the placement-generation logic in
> `mlops-utils/examples/aqcat/adapter.py`, and the Billing section of `mlops-utils/SUMMARY.md`, in
> `sandbox-quantum/external-buildingblocks-collab`.
