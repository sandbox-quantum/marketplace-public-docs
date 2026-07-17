# Adsorption Workflow — Usage Information

## What it does

The adsorption workflow (`mode: "min-adsorption-energy-workflow"`) predicts adsorption energies
for catalysis screening. Given a bulk material (by ID or composition), a surface facet, and an
adsorbate, it builds the surface, generates and relaxes several adsorbate placements, and returns
the relaxed energy and quality flags for each placement. The lowest energy across placements for a
combination is the predicted binding configuration.

The same endpoint also serves `mode: "mlp"` (the default), a raw forward pass that returns energy
and forces for a supplied geometry. If you are looking for min adsorption calculation with parameters that exceed what is supported by the `min-adsorption-energy-workflow` mode, you can use `mlp` mode directly with a client-side script. See [the MLP model docs](../mlp-model/).

## Use cases

- Catalyst screening — rank bulk/facet/adsorbate combinations to find strong binders for a target
  reaction (CO oxidation, hydrogen evolution, oxygen reduction, …).
- Facet comparison — compare how an adsorbate binds across `[1,1,1]`, `[1,0,0]`, etc.
- Adsorbate comparison — screen key intermediates (`*CO`, `*OH`, `*O`, `*H`) on the same surface.
- Composition sweeps — use `bulk_composition_contains` to screen every catalog material containing
  a given element.
- Pre-filtering before DFT — discard weak or unstable candidates before committing
  first-principles compute.

## How to call it

The adsorption workflow should generally be used as a real-time endpoint. A single request can hold many `instances`, so screen several
combinations by batching them into one request rather than sending them one at a time. For large
screens, split the work across multiple smaller requests: a request blocks until every placement
finishes, and a real-time invocation must return within about 60 seconds.

## Instance types

**`ml.g5.xlarge`** (A10) is the recommended instance (the same as MLP mode). The workflow is
CPU-bound rather than GPU-bound, so a higher-end GPU rarely helps. `ml.g6.xlarge`, `ml.g6e.xlarge`,
and `ml.g4dn.xlarge` are also allowed. GPU inference is serialized internally, so concurrent requests
queue rather than fail. For the allowed list, recommended batch sizes, and specs/pricing, see the
[hardware selection guide](../hardware_selection_guide.md) and the
[AWS SageMaker pricing page](https://aws.amazon.com/sagemaker/ai/pricing/).

## Limitations

- Inputs come from the model's bulk and adsorbate catalog (`bulk_src_ids` / `bulk_ids` /
  `bulk_composition_contains`); this mode does not accept arbitrary atomic coordinates.
- Structures are capped at 200 atoms, and composition caps apply (C ≤ 17 %, H ≤ 36 %, O ≤ 23 %,
  N ≤ 17 %).
- Guardrails are enforced, not clamped: `num_placements` 1–5, `max_steps` 1–200, and at most
  1000 instances per request. Violations return `422`. See
  [guardrails](schema.md#guardrails).
- A request body is limited to 6 MB and a real-time invocation to roughly 60 seconds; size
  requests accordingly.
- Results are screening signals, not guarantees. Treat placements with `is_relaxed: false` or any
  failure flag (`is_desorbed`, `is_dissociated`, `is_intercalated`, `is_surface_changed`,
  `is_anomalous`) as unreliable.
- `energy` is the model's relaxed total energy in eV (range −50 to +10), not a referenced
  adsorption energy. Confirm the reference convention before comparing against your DFT.

## Latency

A request blocks until all placements across all instances finish relaxing. Wall-clock time scales
with the number of relaxations (bulks × facets × adsorbates × `num_placements`) and up to
`max_steps` optimizer steps per relaxation. GPU inference is serialized, so overlapping requests
queue.

## Cost

Each successful response is metered as `ConsumedUnits` = the number of model forward passes the
run made (one per optimizer step, summed over all relaxations); failed requests are not billed.
SageMaker instance time bills separately while the endpoint is running, so delete the endpoint and
model when finished. 

For the metering formula and a worked example, see the
[billing guide](../billing_guide.md).
