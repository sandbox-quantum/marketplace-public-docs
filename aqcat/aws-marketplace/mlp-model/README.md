# MLP Mode — Usage Information

## What it does

MLP mode (`mode: "mlp"`, the default) predicts the **total potential energy** (in eV) and the **force on every atom** (in
eV/Å) for an atomic structure you supply — a molecule or a periodic cell. Given an atomic structure as an ASE `Atoms` object (`symbols`, `positions`, `cell`,
`pbc`), it returns the structure's total potential energy (eV) and per-atom forces (eV/Å) from a
single forward pass — a fast surrogate for a DFT single-point calculation. It does in
milliseconds what a quantum-chemistry single-point calculation does in minutes to hours, using
the same FairChem EquiformerV2 + FiLM model that powers the adsorption workflow.

Because it returns energy and forces for *any* geometry, it's the building block for
**optimization and dynamics**: drive your own optimizer or molecular-dynamics loop, calling the
endpoint once per step. A lower (more negative) energy means a more stable structure; when all
forces are near zero, the structure is at an energy minimum.

If your question is "how strongly does this adsorbate bind to this surface?", use the
[adsorption workflow](../adsorption-workflow/) instead — it builds the surface and relaxes it
for you.

## Use cases

- Geometry optimization — drive your own optimizer (e.g. ASE `BFGS`/`LBFGS`) with the returned
  forces, calling the endpoint once per step.
- Molecular dynamics — evaluate energy and forces at each MD step.
- Single-point energies — score a fixed geometry (molecule or periodic cell) without relaxation.
- Custom screening — rank candidate structures by energy when your inputs are raw coordinates
  rather than the bulk/facet/adsorbate combinations the workflow expects.

If the question is "how strongly does this adsorbate bind to this surface?", use the adsorption
workflow instead — it does the surface construction, placement, and relaxation for you.

## How to call it

Use the real-time endpoint. A single request can hold many `instances`, each scored with its own
forward pass, so score several structures by batching them into one request. Keep batch sizes
within GPU memory; see the
[hardware selection guide](../hardware_selection_guide.md) for limits.

## Instance types

**`ml.g5.xlarge`** (A10) is the recommended instance for MLP mode, as it gives the best
cost-to-throughput. Use `ml.g6.xlarge` (L4) if you lack g5 quota, `ml.g6e.xlarge` (L40S) for large
batches or systems near/over ~200 atoms, or `ml.g4dn.xlarge` (T4) as a lowest-cost baseline. GPU
inference is serialized internally, so concurrent requests queue rather than fail. For the full
list of allowed instances, batch-size guidance, and specs/pricing, see the
[hardware selection guide](../hardware_selection_guide.md) and the
[AWS SageMaker pricing page](https://aws.amazon.com/sagemaker/ai/pricing/).

## Limitations

- Single point only. Each call evaluates the geometry you send; the optimization or dynamics loop
  runs on your side.
- Output is energy and forces only — no stress tensor, Hessians, dipoles, or excited-state
  properties.
- Input is a serialized ASE `Atoms` object: `symbols`, `positions`, `cell`, and `pbc` are all
  required (use an all-zeros `cell` with `pbc: [false, false, false]` for an isolated molecule).
  `tags` and `constraints` are accepted but not used, and the spin/fidelity context is set
  internally. The older `numbers` format is rejected with `422`.
- 62 supported elements and a 200-atom cap per structure (see
  [guardrails](schema.md#guardrails)); the workflow's composition caps and
  periodicity requirement do not apply. Accuracy is only meaningful within the model's training
  domain (heterogeneous catalysis) — an in-bounds but out-of-domain request still returns numbers.
- `energy` is reported on the model's internal reference, not as a formation or binding energy.
  Compare energies across geometries of the same system rather than reading them in isolation.
- A request body is limited to 6 MB and a real-time invocation to roughly 60 seconds. MLP mode has
  no per-request instance cap; a request is bounded only by the payload limit.

## Latency

A forward pass is fast, and latency scales mainly with atom count — small molecules typically
return well under a second on GPU. GPU inference is serialized, so overlapping requests queue.

## Cost

Each successful response is metered as `ConsumedUnits` = the number of instances in the request
(one forward pass each); failed requests are not billed. The adsorption workflow meters
differently, by total forward passes across its relaxations. SageMaker instance time bills
separately while the endpoint is running, so delete the endpoint and model when finished. 

For more details, see the [billing guide](../billing_guide.md).
