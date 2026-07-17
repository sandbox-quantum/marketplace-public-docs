# AQCat AWS Marketplace Model Documentation

**AQCat** is a high-fidelity, spin-aware machine learning interatomic potential family that delivers DFT-level adsorption energy accuracy for industrially relevant catalysts at up to 20,000x the speed of physics-based calculations. It is available in AWS Marketplace as a SageMaker model package, which you can deploy to your own tenant for peace-of-mind that your workloads will be secure. Its main use is for catalysis screening.

AQCat is currently built on a [FairChem EquiformerV2](https://github.com/atomicarchitects/equiformer_v2) + FiLM machine-learning potential. You can learn more about AQCat, the ways it is differentiated, and what it's useful for [here](https://www.sandboxaq.com/aqcat25).

The model package exposes a single synchronous endpoint (`POST /invocations`). The request field `mode` selects what it does: 
- `mlp` (the default) runs a raw single-point forward pass of the model, returning energy + forces for a supplied geometry
- `min-adsorption-energy-workflow` is a dense adsorption workflow for heterogeneous catalyst pre-screening, estimating the minimum adsorption energy given a bulk material, a surface facet, and an adsorbate (e.g. `*CO`)

## Start here

| Doc | What's inside |
|--------|---------------|
| [`getting-started`](./getting_started.md) | Clone-and-run listing demo. One notebook that subscribes → deploys → runs → cleans up. The fastest way to try the product end-to-end. |
| [`mlp-model/`](mlp-model/) | The forward pass (`mode:"mlp"`, default) — supply an ASE `Atoms` object (`symbols` + `positions` + `cell` + `pbc`), get back `energy` + `forces`. |
| [`adsorption-workflow/`](adsorption-workflow/) | The adsorption-energy workflow (`mode:"min-adsorption-energy-workflow"`). |
| [`hardware_selection_guide.md`](hardware_selection_guide.md) | GPU instance recommendations. Memory estimates, batch sizes for A10/L4/L40S, and MLP vs adsorption-workflow bottlenecks. |

## What makes AQCat different

- **Spin-aware.** AQCat accounts for **magnetism**: it automatically enables spin polarization
  for systems containing a magnetic element — **Ce, Co, Cr, Cu, Fe, Mn, Mo, Ni, Os, Ru, V, W** —
  and runs spin-unpolarized otherwise. The choice is made for you and reported per result via the
  `is_spin_off` flag (`false` = spin polarization on). This matters for the transition-metal
  catalysts where spin meaningfully changes adsorption energies.
- **Near-DFT accuracy, a fraction of the cost.** A single forward pass of the FairChem
  EquiformerV2 + FiLM potential stands in for a DFT calculation, turning minutes-to-hours of
  compute into milliseconds.
- **End-to-end workflow, not just a potential.** The adsorption mode automates the whole
  pre-screening pipeline — bulk selection → slab generation → adsorbate placement → relaxation —
  behind one API call, with no custom scripting.
- **Broad chemistry.** **62 supported elements** (listed in each mode's
  [schema doc](mlp-model/schema.md)) and, for the adsorption workflow, a large
  built-in catalog of bulk materials and adsorbates selectable by composition.

## Need support?
Email `support.aisim (at) sandboxaq.com` for any bug reports, feature requests, or general support questions.
