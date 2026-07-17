# MLP Mode — Input / Output Schema

This document describes every field accepted and returned by the **MLP (machine-learning
potential) mode** of the **AQCat** model package.

The model exposes **one synchronous endpoint**, `POST /invocations`, plus a `GET /ping` health
check. Request and response bodies are both `application/json`.

---

## Request

```json
{
  "mode": "mlp",
  "instances": [
    {
      "symbols": ["H", "H"],
      "positions": [
        [0.0, 0.0, 0.0],
        [0.74, 0.0, 0.0]
      ],
      "cell": [[0.0, 0.0, 0.0], [0.0, 0.0, 0.0], [0.0, 0.0, 0.0]],
      "pbc": [false, false, false]
    }
  ],
  "parameters": {}
}
```

> Each instance is a JSON-serialized **ASE `Atoms`** object — the same primitive the adsorption
> workflow uses internally. For a non-periodic molecule, pass an all-zeros `cell` and
> `pbc: [false, false, false]`.

### Top-level fields

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `mode` | string | Optional | Defaults to `"mlp"` (this mode), so it can be omitted. Set `"min-adsorption-energy-workflow"` to run the adsorption workflow instead, which expects a completely different instance shape. |
| `instances` | array&lt;object&gt; | **Required** | One or more structures to score. Each is evaluated with one independent forward pass; the model returns one prediction per instance, in the same order. |
| `parameters` | object | Optional | Accepted for envelope compatibility with workflow mode. MLP mode has **no tunable parameters** — pass `{}`. |

### Instance object (`MLPInstance`)

Each instance is a single atomic system, given as a serialized ASE `Atoms` object. `symbols` and
`positions` must have matching length N.

| Name | Type | Required | Description | Example |
|------|------|----------|-------------|---------|
| `symbols` | string[] | **Required** | **Chemical symbols** (element strings), one per atom, length N. E.g. `"H"`, `"C"`, `"O"`, `"Cu"`. | `["H", "H"]` |
| `positions` | array&lt;array&lt;float&gt;&gt; | **Required** | Cartesian coordinates in **Å**, shape `(N, 3)`, in the same order as `symbols`. | `[[0,0,0],[0.74,0,0]]` |
| `cell` | array&lt;array&lt;float&gt;&gt; | **Required** | 3×3 unit-cell vectors in Å. Use all-zeros `[[0,0,0],[0,0,0],[0,0,0]]` for an isolated (non-periodic) molecule. | `[[0,0,0],[0,0,0],[0,0,0]]` |
| `pbc` | bool[] | **Required** | Three booleans — periodicity per axis. `[false, false, false]` for a molecule; `[true, true, true]` for a bulk/slab (set alongside a real `cell`). | `[false, false, false]` |
| `tags` | int[] \| null | Optional | ASE atom tags, length N. Accepted for `Atoms`-format consistency but **not used** by the forward pass. | `[0, 0]` |
| `constraints` | object \| null | Optional | ASE constraints, e.g. `{"FixAtoms": [0, 1]}`. Accepted for format consistency but **not applied** — a single-point forward pass returns raw forces regardless. | `{"FixAtoms": [0]}` |

> **Chemical symbols, not atomic numbers.** MLP mode takes element strings in `symbols` (an ASE
> `Atoms` object), not integer atomic numbers. A water molecule is `"symbols": ["O", "H", "H"]`.

> **Internal FiLM context.** The model's spin/fidelity context (`is_spin_off` / `is_low_fi`) is
> **decided internally** and is no longer part of the public input — there is nothing to set. If a
> request happens to include them, they are **ignored, not rejected**.

---

## Response

```json
{
  "predictions": [
    {
      "energy": -6.7823,
      "forces": [
        [0.0412, 0.0, 0.0],
        [-0.0412, 0.0, 0.0]
      ]
    }
  ]
}
```

One element in `predictions` per input instance, in the same order. Each is an `MLPResult`:

| Name | Type | Description | Example |
|------|------|-------------|---------|
| `energy` | float | Predicted total potential energy of the system, in **eV**. | `-6.7823` |
| `forces` | array&lt;array&lt;float&gt;&gt; | Per-atom forces in **eV/Å**, shape `(N, 3)`, in the same atom order as the input `symbols`. These are the negative gradient of the energy (−∇E): the force on an atom points in the direction that lowers the energy, and near-zero forces mean the geometry is at a local minimum. | `[[0.0412,0,0],[-0.0412,0,0]]` |

> MLP mode returns **only** `energy` and `forces`. It does **not** return stress, the relaxed
> geometry, anomaly flags, or a `combo_key` — those belong to the adsorption workflow. To
> minimise energy you run your own optimizer (e.g. ASE) and call the endpoint at each step.

---

## Errors

Failures return a non-2xx status with an error body:

```json
{ "error": "<message>" }
```

| Code | Meaning |
|------|---------|
| `422` | Schema violation in an `instances` entry — a missing required field (`symbols`/`positions`/`cell`/`pbc`), a `symbols`/`positions` length mismatch, an unsupported element, exceeding the atom-count cap, or the **old decomposed `numbers` format** (no longer accepted). Also returned if `mode` is routed to an adapter that doesn't support it. |
| `500` | Runtime error during the forward pass. |

---

## Guardrails

MLP mode applies two input constraints (it reuses the adsorption workflow's `_validate_system`),
and rejects violations with `422`:

- **Supported elements (62).** Every `symbols` entry must be one of:

  > Ag, Al, As, Au, B, Ba, Bi, C, Ca, Cd, Ce, Cl, Co, Cr, Cs, Cu, F, Fe, Ga, Ge, H, Hf, Hg, In,
  > Ir, K, La, Li, Mg, Mn, Mo, N, Na, Nb, Ni, O, Os, P, Pb, Pd, Pt, Rb, Re, Rh, Ru, S, Sb, Sc, Se,
  > Si, Sn, Sr, Ta, Tc, Te, Ti, Tl, V, W, Y, Zn, Zr

- **Atom count.** 1–**200** atoms per structure (`MAX_ATOM_COUNT = 200`).

The workflow's **composition caps** (C/H/O/N %) and its **mandatory-periodicity** check do **not**
apply to MLP — so ordinary molecules like water and methane (non-periodic, all-zeros `cell`) are
accepted. There is also no per-request **instance** cap (unlike the workflow's 1000); a request is
bounded only by the 6 MB payload. Within the supported element set, accuracy is still only
meaningful for chemistry inside the model's training distribution (heterogeneous-catalysis
systems); a valid request far outside that domain will return numbers that may not be physically
reliable.

> **Hard cut from the old schema.** The previous decomposed input (`numbers` + `positions` +
> optional `cell`/`pbc`/`is_spin_off`/`is_low_fi`) is **no longer accepted** — a payload using
> `numbers` instead of an `Atoms` object (`symbols`/`positions`/`cell`/`pbc`) returns `422`.

The transport limits still apply: a request body is **≤ 6 MB**, and a **real-time** invocation
must respond within **~60 s**.

---

## Batch transform

> ⚠️ **Not recommended.** SandboxAQ does not actively support batch transform jobs, and recommend batching available via real-time inference.

The schema is identical for batch transform. Each input file is **one complete request body**
(`{ "mode": "mlp", "instances": [...], "parameters": {} }`) and may contain many instances. The
model processes one file per request (`BatchStrategy=SingleRecord`, `SplitType=None`,
`ContentType=application/json`, ≤ 6 MB), writing one `*.out` file containing the `predictions`
JSON. See [`../sample_data/input_batch_sample.json`](../sample_data/input_batch_sample.json) and
[`../sample_data/output_batch_sample.json`](../sample_data/output_batch_sample.json).

### Sizing & limits

- **Record granularity.** `SingleRecord` + `SplitType=None` → **one file = one request body**
  (which may hold many instances). Don't concatenate multiple bodies into one file.
- **Per-file payload:** ≤ **6 MB** (`MaxPayloadInMB`). MLP instances are compact, so this holds many
  structures; very large geometries (many atoms × positions) consume more, so split across files if
  a body approaches 6 MB.
- **Instances per request:** **no explicit cap** in MLP mode (unlike workflow's 1000) — bounded only
  by the 6 MB payload.
- **Parallelism / "batch size":** throughput comes from many files + a higher transform
  `InstanceCount`; there is no mini-batch knob, and GPU work is serialized internally
  (`MAX_PARALLEL`).

---

## Billing

Successful (2xx) responses carry the AWS Marketplace metering header:

```
X-Amzn-Inference-Metering: {"Dimension":"inference.count","ConsumedUnits":N}
```

For MLP mode, `ConsumedUnits` = the **number of instances** in the request (one unit per
forward pass). This differs from workflow mode, which meters by the **total model forward passes**
across all relaxations. Failed (4xx/5xx) responses are **not** billed.

See more in the [billing guide](../billing_guide.md).
