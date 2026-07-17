# AQCat — Input / Output Schema

This document describes every field accepted and returned by the **AQCat** model package.
Given a **bulk material + surface facet + adsorbate**, the model builds the surface, places the
adsorbate, relaxes the structure, and returns the relaxed **energy** (and quality flags) for each
placement — a fast surrogate for a DFT adsorption-energy calculation.

The model exposes **one synchronous endpoint**, `POST /invocations`, plus a `GET /ping` health
check. There is no asynchronous/job path. The request and response bodies are both
`application/json`.

---

## Request

```json
{
  "mode": "min-adsorption-energy-workflow",
  "instances": [
    {
      "bulk_src_ids": ["mp-102"],
      "facets": [[1, 1, 1]],
      "adsorbates": ["*CO"],
      "num_placements": 3
    }
  ],
  "parameters": {
    "seed": 42,
    "fmax": 0.05,
    "max_steps": 150,
    "converged_placements_only": true
  }
}
```

### Top-level fields

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `mode` | string | **Required for this workflow** | Set to `"min-adsorption-energy-workflow"`. The endpoint's default mode is `"mlp"`, so if `mode` is omitted the request runs the raw forward pass instead and rejects this instance shape. (The legacy value `"workflow"` is still accepted as an alias.) |
| `instances` | array&lt;object&gt; | **Required** | One or more screening requests. Each instance is one bulk/facet/adsorbate combination. A single request may carry many instances. |
| `parameters` | object | Optional | Relaxation settings shared by all instances. See *Parameters* below. |

### Instance object

Each instance must set **exactly one** bulk selector, plus facets, adsorbates, and a placement count.

| Name | Type | Required | Description | Example |
|------|------|----------|-------------|---------|
| `bulk_src_ids` | string[] | one selector | Bulk material IDs from the source catalog (Materials Project-style IDs). | `["mp-102"]` |
| `bulk_ids` | int[] | one selector | Bulk materials by internal integer ID. | `[72]` |
| `bulk_composition_contains` | string | one selector | Select bulks whose composition contains an element. | `"Cu"` |
| `facets` | array&lt;array&lt;int&gt;&gt; | **Required** | Surface facets as Miller indices `[h, k, l]`. | `[[1, 1, 1]]` |
| `adsorbates` | string[] | one selector | Adsorbate species in catalysis SMILES notation (leading `*` = adsorbed). | `["*CO"]` |
| `adsorbate_ids` | int[] | one selector | Adsorbates by internal database ID, as an alternative to `adsorbates`. | `[10]` |
| `num_placements` | int | Optional | How many adsorbate placements to generate and relax **per (bulk × facet × adsorbate) combination**. Default `5`, range **1–5**. | `3` |

> Each instance must set **exactly one** bulk selector (`bulk_src_ids` / `bulk_ids` /
> `bulk_composition_contains`) **and exactly one** adsorbate selector (`adsorbates` /
> `adsorbate_ids`).

### Parameters object

Shared by all instances in the request. All are optional; the defaults below are applied when a
field is omitted.

**Relaxation**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `fmax` | float | Force-convergence threshold for the relaxation (eV/Å). Relaxation stops when the max force on any atom drops to `fmax`. | `0.05` |
| `max_steps` | int | Max relaxation steps per placement. Range **1–200**. | `150` |
| `seed` | int | RNG seed for placement generation. The same seed reproduces identical placements (and energies) across runs. Omit for non-deterministic placement. | `null` |
| `calc_kwargs` | object | Extra keyword arguments forwarded to the underlying calculator. Leave empty unless advised. | `{}` |

**Slab & placement**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `max_bulks` | int | When selecting bulks by composition, the maximum number of matching bulks to use. | `1` |
| `placement_mode` | string | Algorithm for generating adsorbate placements on the slab. | `"random_site_heuristic_placement"` |
| `interstitial_gap` | float | Gap (Å) inserted between slab and adsorbate when placing. | `0.1` |
| `min_ab` | float | Minimum in-plane (x, y) slab dimensions in Å. | `8.0` |

**FiLM context flags**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `is_spin_off` | bool | Override the model's FiLM **spin** context. If omitted, auto-detected from the element set. | `null` (auto) |
| `is_low_fi` | bool | Override the model's FiLM **fidelity** context. | `false` |

**Bulk resolution**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `bulk_composition_exact_only` | bool | When `true`, a formula-shaped `bulk_composition_contains` requires an exact stoichiometric match in the catalog; otherwise the request errors (no fuzzy fallback). | `true` |

**Output filtering**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `converged_placements_only` | bool | If `true`, drop placements that did not reach `fmax` within `max_steps`. Default `false` keeps them — filter client-side on `is_relaxed`. | `false` |
| `filter_anomalous_placements` | bool | If `true` (default), drop placements flagged anomalous (desorbed / dissociated / intercalated / surface-changed) — their energies aren't meaningful against the intended adslab reference. | `true` |
| `metadata_only` | bool | If `true` (default), omit the heavy per-atom `atoms` dict from each prediction; the metadata fields (`energy`, flags, `traj_file`, …) remain. Set `false` to include full relaxed structures. | `true` |
| `desorption_cutoff_multiplier` | float | Cutoff multiplier for desorption detection (`DetectTrajAnomaly`). Lower flags more placements as anomalous. | `1.5` |
| `surface_change_cutoff_multiplier` | float | Cutoff multiplier for surface-reconstruction detection. Lower flags more placements as anomalous. | `1.5` |

---

## Response

```json
{
  "predictions": [
    {
      "combo_key": "CO_mp-102_111_167_True",
      "placement_idx": 0,
      "adsorbate": "*CO",
      "energy": -7.23,
      "max_force": 0.04,
      "is_relaxed": true,
      "is_spin_off": false,
      "is_low_fi": false,
      "is_anomalous": false,
      "is_desorbed": false,
      "is_dissociated": false,
      "is_intercalated": false,
      "is_surface_changed": false,
      "traj_file": "trajectories/CO_mp-102_111_167_True_p000.traj"
    }
  ]
}
```

One element in `predictions` per placement. Placements for the same bulk/facet/adsorbate share a
`combo_key`.

| Name | Type | Description | Example |
|------|------|-------------|---------|
| `combo_key` | string | Identifier grouping all placements for one bulk/facet/adsorbate combination. | `"CO_mp-102_111_167_True"` |
| `placement_idx` | int | Zero-based, sequential index of the placement within its `combo_key`. | `0` |
| `adsorbate` | string | Adsorbate species, echoing the request. | `"*CO"` |
| `energy` | float | Relaxed total energy in **eV**. Always finite, in the range **(−50, +10)**. Lower = more stable; the lowest energy across placements is the predicted binding configuration. | `-7.23` |
| `max_force` | float | Largest residual force on any atom after relaxation (≥ 0). Near `fmax` indicates a converged structure. | `0.04` |
| `is_relaxed` | bool | Whether the relaxation converged (reached `fmax` within `max_steps`). | `true` |
| `is_spin_off` | bool | Quality flag — spin handling fell back. | `false` |
| `is_low_fi` | bool | Quality flag — low-fidelity result. | `false` |
| `is_anomalous` | bool | Quality flag — result flagged as anomalous. | `false` |
| `is_desorbed` | bool | Adsorbate detached from the surface during relaxation. | `false` |
| `is_dissociated` | bool | Adsorbate broke apart during relaxation. | `false` |
| `is_intercalated` | bool | Adsorbate migrated into the bulk. | `false` |
| `is_surface_changed` | bool | The surface reconstructed significantly. | `false` |
| `traj_file` | string | Path to the relaxation trajectory (`trajectories/*.traj`). | `"trajectories/CO_mp-102_111_167_True_p000.traj"` |
| `slab_id` | string \| null | Identifier of the slab the adsorbate was placed on. | `"mp-102_111_167_True"` |
| `atoms` | object \| null | Full serialized relaxed structure (positions, cell, symbols, forces, …). Omitted (`null`) unless `metadata_only` is set to `false` in the request parameters. | `null` |

> The boolean `is_*` flags are **screening quality signals** — a physically meaningful binding
> energy is one where `is_relaxed` is `true` and the failure flags (`is_desorbed`,
> `is_dissociated`, `is_intercalated`, `is_surface_changed`, `is_anomalous`) are all `false`.

> **Field notes:** `max_force` is in **eV/Å**. `is_spin_off` and `is_low_fi` are the model's
> **FiLM context flags** — `is_spin_off` reflects the magnetic-spin context, auto-detected from the
> element set: spin polarization is **enabled** (`is_spin_off: false`) when the structure contains
> any magnetic element (**Ce, Co, Cr, Cu, Fe, Mn, Mo, Ni, Os, Ru, V, W**), and disabled otherwise.
> `is_low_fi` reflects the fidelity context. `slab_id` and the full `atoms` structure are returned
> per the request's `metadata_only` setting (see *Parameters*).

---

## Errors

Failures return a non-2xx status with an error body:

```json
{ "error": "<message>" }
```

Adapter-level failures use the `{ "error": "<message>" }` body above; envelope-level validation
(a malformed request body) is FastAPI's automatic `422` with a `detail[]` array instead. Status
codes emitted by the container:

| Code | Meaning |
|------|---------|
| `422` | Schema or guardrail violation — bad instance schema, `num_placements` > 5, `max_steps` > 200, or too many instances (> 1000). Never silently clamped. |
| `500` | Runtime error during inference. |

> The container does not emit `400`. A real-time invocation may still receive a SageMaker
> **endpoint-level** `504` if the request exceeds the ~60 s response limit — that comes from the
> hosting layer, not the model.

---

## Batch transform

> ⚠️ **Not recommended.** SandboxAQ does not actively support batch transform jobs, and recommend using batching available via real-time inference.

The schema is identical for batch transform. Each input file is **one complete request body**
(`{ "instances": [...], "parameters": {...} }`) and may contain multiple instances. The model
processes one file per request (`BatchStrategy=SingleRecord`, `SplitType=None`,
`ContentType=application/json`, ≤ 6 MB), writing one `*.out` file containing the `predictions`
JSON. See [`../sample_data/input_batch_sample.json`](../sample_data/input_batch_sample.json) and
[`../sample_data/output_batch_sample.json`](../sample_data/output_batch_sample.json).

### Sizing & limits

- **Record granularity.** `SingleRecord` + `SplitType=None` means **one file = one request body**.
  Do not concatenate multiple JSON bodies into a file; put one body per file.
- **Per-file payload:** ≤ **6 MB** (`MaxPayloadInMB`). The JSON is compact (each instance is a few
  short fields), so 6 MB comfortably holds the per-request instance cap below.
- **Instances per request:** at most **1000** (`422 Too many instances` beyond that). This applies
  to both real-time and batch, since a batch file is one request body.
- **Parallelism / "batch size":** there is no mini-batching knob — throughput comes from running
  **many files** and raising `InstanceCount` on the transform job (and/or `MaxConcurrentTransforms`).
  Per instance, GPU work is serialized internally (`MAX_PARALLEL`), so requests queue rather than
  fail.
- **Long jobs belong in batch.** A real-time invocation has a hard **~60-second** response limit and
  a **6 MB** request limit; an adsorption relaxation with large `num_placements × max_steps` can
  exceed 60 s and return a `504`.

---

## Guardrails

Enforced **before** inference; violations are rejected (not clamped).

| Parameter | Default | Min | Max | On violation |
|-----------|---------|-----|-----|--------------|
| `num_placements` | 5 | 1 | 5 | `422` |
| `max_steps` | 150 | 1 | 200 | `422` |
| `instances` (per request) | — | 1 | **1000** | `422` ("Too many instances") |

> `num_placements` (≤ 5) and `max_steps` (≤ 200) are capped to keep a request within the
> real-time endpoint's **~60 s** response limit. Note this is a per-parameter cap, not a guarantee:
> total work is `bulks × facets × adsorbates × num_placements` relaxations × up to `max_steps`
> steps, so a request with many facets/adsorbates/instances can still approach the limit — size
> requests accordingly.

System-level limits (enforced in the adapter on the **generated adslab**): maximum **200 atoms**
per structure, and composition caps (C > 17 %, H > 36 %, O > 23 %, N > 17 %). Every element in the
selected bulk and adsorbate must be one of the **62 supported elements**:

> Ag, Al, As, Au, B, Ba, Bi, C, Ca, Cd, Ce, Cl, Co, Cr, Cs, Cu, F, Fe, Ga, Ge, H, Hf, Hg, In, Ir,
> K, La, Li, Mg, Mn, Mo, N, Na, Nb, Ni, O, Os, P, Pb, Pd, Pt, Rb, Re, Rh, Ru, S, Sb, Sc, Se, Si,
> Sn, Sr, Ta, Tc, Te, Ti, Tl, V, W, Y, Zn, Zr

Transport limits: a request body is ≤ 6 MB and a real-time invocation must respond within ~60 s.

---

## Billing

Successful (2xx) responses carry a metering header for AWS Marketplace:

```
X-Amzn-Inference-Metering: {"Dimension":"inference.count","ConsumedUnits":N}
```

`ConsumedUnits` = the number of **model forward passes (MLP calls)** the run made — one per
optimizer (LBFGS) step, summed over every relaxation. A request runs
`bulks × facets × adsorbates × num_placements` relaxations, and each relaxation takes **up to
`max_steps`** steps (fewer if it converges to `fmax` first), so
`ConsumedUnits ≤ (bulks × facets × adsorbates × num_placements) × max_steps`. It is **not** a flat
`num_placements` count. Failed (4xx/5xx) responses are **not** billed.

See more in the [billing guide](../billing_guide.md).
