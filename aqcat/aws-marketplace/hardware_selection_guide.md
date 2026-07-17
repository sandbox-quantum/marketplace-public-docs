# AQCat Hardware Selection Guide

This guide provides recommendations for selecting **GPU instance types** on SageMaker based on your
workload. The same endpoint serves both modes.

| `mode` | What it does | Additional docs |
|---|---|---|
| `"mlp"` (default) | Raw forward pass — atoms in, energy + forces out | [`mlp-model/`](mlp-model/) |
| `"min-adsorption-energy-workflow"` | Build surface → place adsorbate → relax → adsorption energies | [`adsorption-workflow/`](adsorption-workflow/) |

The **benchmarked recommendations below** use the **g5** and **g6** families
for better cost and throughput when your account has quota for them.

---

## Allowed instance types

| Instance | GPU | Recommended batch size | Use it for |
|---|---|---|---|
| `ml.g5.xlarge` | A10 | 6 | **Recommended** for MLP and the adsorption workflow — best cost-to-throughput |
| `ml.g6.xlarge` | L4 | 6 | Alternative when you lack g5 quota or get better regional pricing |
| `ml.g6e.xlarge` | L40S | 13 | Large batches, or systems near/over ~200 atoms (high VRAM) |
| `ml.g4dn.xlarge` | T4 | 4 | Lowest-cost baseline for small or low-volume jobs |

Specs and pricing are on the
[AWS SageMaker pricing page](https://aws.amazon.com/sagemaker/ai/pricing/).

---

## `mode: "mlp"` — direct model inference

In MLP mode, performance is primarily determined by the **model inference** step.

### Memory requirements

The approximate GPU memory footprint is:

```
mem(N) ≈ 217 + 40 × N   (MB)
```

where **N** is the **total number of atoms in a batch**.

### Recommended batch sizes

For systems with a moderate number of atoms (approximately **60–85 atoms per structure**):

| GPU | SageMaker instance | Recommended max batch size |
|-----|-------------------|---------------------------|
| T4 | `ml.g4dn.xlarge` | 4 |
| A10 | `ml.g5.xlarge` | 6 |
| L4 | `ml.g6.xlarge` | 6 |
| L40S | `ml.g6e.xlarge` | 13 |

### Cost and performance recommendations

- The **A10** (`ml.g5.xlarge`) generally provides the best **cost-to-throughput ratio**.
- If your workload consists of **large batches of moderately sized systems**, it is usually more
  cost-effective to use an **A10** rather than moving to a larger GPU.
- Larger GPUs (e.g. **L40S** on `ml.g6e.xlarge`) primarily become beneficial when the memory
  requirements of your workload exceed the capacity of smaller GPUs.

### Large systems

For very large simulations, or for batches containing structures with **more than ~200 atoms**,
estimate the required memory using:

```
mem(N) ≈ 217 + 40 × N   (MB)
```

where **N** is the total number of atoms in the batch.

Choose a GPU with sufficient memory capacity to accommodate the estimated footprint while leaving
some additional headroom.

> MLP mode does **not** enforce the workflow's 200-atom cap, but memory use and accuracy both grow
> with system size. See [`mlp-model/README.md`](mlp-model/README.md).

---

## `mode: "min-adsorption-energy-workflow"` — adsorption screening

In adsorption workflow mode, runtime is typically dominated by the conventional **LBFGS energy
minimization** step rather than by the MLP inference pass.

As a result:

- Performance is generally **CPU-bound**, not GPU-bound.
- Increasing GPU memory or upgrading to a larger GPU will usually provide **little to no speedup**.
- Selecting a larger GPU is only recommended if the system does not fit into the memory available
  on the current GPU.

To determine whether a larger GPU is required, use the same memory estimate:

```
mem(N) ≈ 217 + 40 × N   (MB)
```

where **N** is the total number of atoms in a batch.

For adsorption workflow mode, the recommended instance is **`ml.g5.xlarge`** (A10) — the same as MLP.
The workflow is CPU-bound rather than GPU-bound, so a higher-end GPU rarely helps; only move up a GPU
tier if a structure does not fit in memory.

> Workflow mode caps structures at **200 atoms** and meters by total `num_placements`. See
> [`adsorption-workflow/README.md`](adsorption-workflow/README.md).

---

## Quick reference

| Your workload | Recommended starting point | Notes |
|---|---|---|
| MLP — moderate structures (~60–85 atoms), cost-sensitive | `ml.g5.xlarge` (A10) | Best cost-to-throughput; batch up to **6** |
| MLP — same size, g6 quota | `ml.g6.xlarge` (L4) | Comparable batch size (**6**); check regional pricing |
| MLP — larger batches or heavier atom counts | `ml.g6e.xlarge` (L40S) | Higher VRAM; batch up to **13** for moderate structures |
| Adsorption workflow — typical screens | `ml.g5.xlarge` (A10) | Recommended; the workflow is CPU-bound, so a bigger GPU rarely helps |
| Adsorption — structure near memory limit | Next GPU tier per `mem(N)` estimate | Only upgrade GPU if the structure does not fit |
