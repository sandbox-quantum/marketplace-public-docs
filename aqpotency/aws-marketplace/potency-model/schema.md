# Potency Mode — Input / Output Schema

This document describes every field accepted and returned by the **potency mode** of the
**AQPotency** model package.

The model exposes **one synchronous endpoint**, `POST /invocations`, plus a `GET /ping` health
check. Request and response bodies are both `application/json`. Every response has the shape
`{"predictions": [...], "warnings": [...]}` and carries the `X-Amzn-Inference-Metering` header.

---

## Request

```json
{
  "mode": "potency",
  "applicability_domain": true,
  "instances": [
    {
      "smiles": "COc1cc2ncnc(Nc3ccc(F)c(Cl)c3)c2cc1OCCCN1CCOCC1",
      "uniprot_id": "P00533"
    }
  ]
}
```

> Each instance is a single `(ligand, target)` pair: a ligand **SMILES** string and a target
> **UniProt accession**. There are no 3D coordinates or structural inputs.

### Top-level fields

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `mode` | string | **Required** | Set to `"potency"`. `mode` is required — a missing or unrecognized value returns `422` (there is no default). |
| `applicability_domain` | bool | Optional | Toggle the applicability-domain computation. Default `true`; set `false` to skip AD for a faster response — the per-prediction `applicability_domain` field then comes back `null`. Does **not** change billing (only speed/infra cost). |
| `instances` | array&lt;object&gt; | **Required** | One or more `(ligand, target)` pairs to score. Each is evaluated independently; the model returns one prediction per instance, in the same order. |

### Instance object (`PotencyInstance`)

| Name | Type | Required | Description | Example |
|------|------|----------|-------------|---------|
| `smiles` | string | **Required** | Ligand structure as a **SMILES** string. Canonicalized (RDKit) server-side before scoring. An unparseable SMILES is rejected with `422`. | `"CCO"` |
| `uniprot_id` | string | **Required** | **UniProt accession** of the protein target. Must be in the model's target vocabulary, or the request is rejected with `422`. | `"P00533"` |

> **SMILES + UniProt, not structures.** Potency mode takes a ligand SMILES and a target UniProt
> accession — not atomic coordinates and not a protein sequence. The target is identified by
> accession and looked up in the model's vocabulary.

---

## Response

```json
{
  "predictions": [
    {
      "smiles": "COc1cc2ncnc(Nc3ccc(F)c(Cl)c3)c2cc1OCCCN1CCOCC1",
      "uniprot_id": "P00533",
      "potency_mean": 7.64,
      "potency_sigma": 1.12,
      "applicability_domain": 0.83,
      "target_name": "EGFR"
    }
  ],
  "warnings": []
}
```

One element in `predictions` per input instance, in the same order. The inputs are **echoed** next to
the scores. Each is a `PotencyResult`:

| Name | Type | Description | Example |
|------|------|-------------|---------|
| `smiles` | string | The scored ligand, echoing the (canonicalized) input. | `"CCO"` |
| `uniprot_id` | string | The target, echoing the input. | `"P00533"` |
| `potency_mean` | float | Predicted potency as **pIC50** = −log₁₀(IC50 in mol/L). **Higher = more potent** (7.0 ≈ 100 nM, 9.0 ≈ 1 nM). | `7.64` |
| `potency_sigma` | float | The model's **uncertainty** on `potency_mean` — a standard deviation in pIC50 units. Smaller = more confident. | `1.12` |
| `applicability_domain` | float \| null | **In-/out-of-domain** score (~0–1): similarity of the input to the ~1.35 M training compounds. High = in-domain/reliable; low = extrapolation. **`null`** when the request set `applicability_domain: false`. | `0.83` |
| `target_name` | string \| null | Resolved human-readable protein name for `uniprot_id`, or `null` if unavailable. | `"EGFR"` |

> **The `warnings[]` array.** A top-level `warnings` list accompanies every response. When an
> input's `applicability_domain` falls below `AD_WARN_THRESHOLD` (default `0.4`), a note is added to
> `warnings[]` — the prediction is still returned (it is **not** an error), but it is flagged as
> likely extrapolation.

> Potency mode returns **only** the four fields above per prediction. The panel `hits[]` /
> `prob_above_threshold` / `alert` fields belong to [`scan`](../scan-workflow/schema.md), and the
> per-compound `results[]` / `n_scored` fields belong to [`screen`](../screen-workflow/schema.md).

---

## Errors

Failures return a non-2xx status with an error body:

```json
{ "error": "<message>" }
```

| Code | Meaning |
|------|---------|
| `422` | Schema or guardrail violation in the request — a missing required field (`smiles`/`uniprot_id`), an **unparseable SMILES**, a `uniprot_id` **not in the target vocabulary**, exceeding the instance cap, or an otherwise malformed payload. |
| `500` | Unexpected runtime error during inference. |

> The container does not emit `400`. A real-time invocation may still receive a SageMaker
> **endpoint-level** `504` if the request exceeds the ~60 s response limit — that comes from the
> hosting layer, not the model. `GET /ping` returns `503` until the model is loaded, `200` after.

---

## Guardrails

Enforced **before** inference; violations are rejected (not clamped) with `422`:

- **Known mode.** `mode` is **required** and must be one of `potency` / `scan` / `screen` /
  `applicability_domain` / `selectivity`; a missing or unknown value → `422`.
- **Target in vocabulary.** Every `uniprot_id` must be a target the model knows; unknown
  accessions are rejected.
- **Parseable SMILES.** Every `smiles` must parse (RDKit); an unparseable SMILES is rejected in
  potency mode. (In [`screen`](../screen-workflow/schema.md) mode, unparseable SMILES are tolerated
  and returned with `null` fields instead.)
- **Instance count.** 1–**15** instances per request (`MAX_POTENCY = 15`). More → `422` (split
  across requests).

**Applicability-domain warning (not a rejection):** an input whose `applicability_domain` is below
`AD_WARN_THRESHOLD` (default `0.4`) still returns a prediction, but adds a note to the response
`warnings[]`.

The transport limits also apply: a request body is **≤ 6 MB**, and a **real-time** invocation must
respond within **~60 s**.

---

## Batch transform

> ⚠️ **Not recommended.** SandboxAQ does not actively support batch transform jobs for interactive
> use, and recommend the batching available via real-time inference. (Batch transform is used
> internally only for model-package validation.)

The schema is identical for batch transform. Each input file is **one complete request body**
(`{ "mode": "potency", "instances": [...] }`) and may contain up to 15 instances. The model
processes one file per request (`BatchStrategy=SingleRecord`, `SplitType=None`,
`ContentType=application/json`, ≤ 6 MB), writing one `*.out` file containing the response JSON. See
[`sample_data/input_realtime_sample.json`](sample_data/input_realtime_sample.json) and
[`sample_data/output_realtime_sample.json`](sample_data/output_realtime_sample.json) for the body
shape.

Batch transform for AQPotency runs on **`ml.m5.2xlarge`**. See the
[hardware selection guide](../hardware_selection_guide.md).

---

## Billing

Successful (2xx) responses carry the AWS Marketplace metering header:

```
X-Amzn-Inference-Metering: {"Dimension":"inference.count","ConsumedUnits":N}
```

For potency mode, `ConsumedUnits` = the **number of predictions** returned (one per instance). This
differs from `scan` (which meters by panel targets scored) and `screen` (by compounds scored).
Failed (4xx/5xx) responses are **not** billed.

See more in the [billing guide](../billing_guide.md).
