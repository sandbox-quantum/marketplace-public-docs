# Screen Workflow — Input / Output Schema

This document describes every field accepted and returned by the **screen mode** of the **AQPotency**
model package. Given a **library of ligands + a single target**, the model scores every compound and
returns the results ranked by predicted potency.

`screen` runs in **two paths, auto-detected by request `Content-Type`**:

| Path | Content-Type | Returns | Use for |
|---|---|---|---|
| Real-time JSON | `application/json` | JSON `{"predictions":[...],"warnings":[...]}` | Small size libraries |
| **CSV / Batch Transform** (**recommended**) | `text/csv` | CSV rows | Large libraries (25k+, millions ...) |

The endpoint is `POST /invocations` (+ `GET /ping` health, `GET /execution-parameters` for Batch
Transform). `mode` is **required** — a missing/unknown value returns `422`. Successful responses carry
the `X-Amzn-Inference-Metering` header.

<br><br>
---
---
<br><br>

## Real-time path (JSON input and output)

### Request

```json
{
  "mode": "screen",
  "applicability_domain": true,
  "instances": [
    {
      "smiles": ["CCO", "c1ccccc1", "COc1cc2ncnc(Nc3ccc(F)c(Cl)c3)c2cc1OCCCN1CCOCC1"],
      "uniprot_id": "P00533"
    }
  ]
}
```

> Each instance screens **one library against one target**: a `smiles` **array** and a single
> `uniprot_id`.

#### Top-level fields

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `mode` | string | **Required** | Set to `"screen"`. Missing/unknown `mode` → `422` (no default). |
| `applicability_domain` | bool | Optional | Default `true`; set `false` to skip the AD pass for a faster response (per-compound `applicability_domain` returns `null`). ~3× faster on large jobs; billing unchanged. |
| `instances` | array&lt;object&gt; | **Required** | One or more screens, each a library vs a single target. |

#### Instance object (`ScreenInstance`)

| Name | Type | Required | Description | Example |
|------|------|----------|-------------|---------|
| `smiles` | string[] | **Required** | The compound **library** — an array of ligand SMILES. Canonicalized (RDKit) server-side; unparseable entries are tolerated (see Response). At most `MAX_SCREEN` (default **25000**) per instance. | `["CCO", "c1ccccc1"]` |
| `uniprot_id` | string | **Required** | **UniProt accession** of the target. Must be in the model's target vocabulary, or `422`. | `"P00533"` |

### Response (JSON)

```json
{
  "predictions": [
    {
      "uniprot_id": "P00533",
      "target_name": "EGFR",
      "n_scored": 2,
      "results": [
        { "smiles": "COc1cc2ncnc(Nc3ccc(F)c(Cl)c3)c2cc1OCCCN1CCOCC1", "potency_mean": 7.64, "potency_sigma": 1.12, "applicability_domain": 0.83 },
        { "smiles": "c1ccccc1", "potency_mean": 4.10, "potency_sigma": 0.42, "applicability_domain": 0.55 }
      ]
    }
  ],
  "warnings": []
}
```

One element in `predictions` per input instance — a `ScreenResult`:

| Name | Type | Description |
|------|------|-------------|
| `uniprot_id` | string | The screened target, echoing the request. |
| `target_name` | string \| null | Resolved protein name, or `null`. |
| `n_scored` | int | Number of compounds **successfully scored** (excludes unparseable SMILES). The billed unit for this instance. |
| `results` | array&lt;object&gt; | Per-compound predictions, sorted by `potency_mean` (desc). |

Each element of `results` is a `ScreenHit`: `smiles` (echoed), `potency_mean` (pIC50),
`potency_sigma`, `applicability_domain` (~0–1). All three score fields are `null` if the SMILES was
unparseable, **or** if `applicability_domain: false` was set (then only the `applicability_domain`
field is `null`).

> **Unparseable SMILES never fail the request** — returned with `null` scores and excluded from
> `n_scored` (and from billing). A compound whose `applicability_domain` is below `AD_WARN_THRESHOLD`
> (default `0.4`) still returns a value but adds a note to `warnings[]`.

### Errors

```json
{ "error": "<message>" }
```

| Code | Meaning |
|------|---------|
| `422` | Schema/guardrail violation — missing `smiles`/`uniprot_id`, a `uniprot_id` **not in the target vocabulary**, a real-time library exceeding `SCREEN_MAX_COMPOUNDS`/`REALTIME_MAX_ENTRIES`, or a batch `screen` CSV that carries more than one distinct target. (Unparseable SMILES do **not** cause `422` — tolerated per-compound.) |
| `500` | Unexpected runtime error during inference. |

> No `400`. A real-time invocation may receive a SageMaker `504` if it exceeds ~60 s. `GET /ping` →
> `503` until loaded, `200` after.


### Guardrails

Enforced before inference; violations rejected (not clamped) with `422`:

- **Known mode.** `mode` required and valid.
- **Target in vocabulary.** `uniprot_id` must be known.
- **Library size (real-time).** ≤ `SCREEN_MAX_COMPOUNDS` (**1000**) per instance; ≤
  `REALTIME_MAX_ENTRIES` (**100000**) total per JSON request. Larger → use the CSV / Batch path.
- **Single target per `screen` CSV** (see above).
- **Unparseable SMILES tolerated** — `null` fields, excluded from `n_scored`.

### Billing

```
X-Amzn-Inference-Metering: {"Dimension":"inference.count","ConsumedUnits":N}
```

`ConsumedUnits` = the **total number of compounds successfully scored** (N = Σ `n_scored`). Unparseable
compounds are not counted. The AD toggle does **not** change units (only speed). Failed (4xx/5xx)
responses are **not** billed. See the [billing guide](../billing_guide.md).

<br><br>
---
---
<br><br>

## Batch Transform (recommended for large libraries)

Recommended for 500k+ compounds (millions and beyond), run a SageMaker **Batch Transform** job with
`Content-Type: text/csv`. It runs the same single-target `screen` as real-time, at CSV/S3 scale, and
writes a scored CSV back to S3.

### The target travels in the CSV (no environment variables)

A Marketplace-subscribed `ModelPackage` **rejects a container `Environment` map** — on **both**
`create_model` and `create_transform_job`. So batch configuration cannot be passed as env under the
conditions a customer runs in. `screen` is **single-target**, so the target travels **in the input
CSV**: a **headerless** file with **column 1 = SMILES, column 2 = the target UniProt ID** (the same ID
on every row). The container reads the target from column 2 — zero env.

### Input CSV (headerless: `SMILES,target`)

```
CCO,P00533
c1ccccc1,P00533
COc1cc2ncnc(Nc3ccc(F)c(Cl)c3)c2cc1OCCCN1CCOCC1,P00533
```

Every row must carry the **same** target; more than one distinct target → `422`.

### Output CSV

Each input row echoed with the target and scores, in input order, in the output S3 bucket:

```
SMILES,uniprot_id,potency_mean,potency_sigma,applicability_domain
CCO,6.49,1.23,P00533,0.085
c1ccccc1,4.10,0.42,P00533,0.550
```

### Parallelism (`InstanceCount` > 1) is file-granular

SageMaker assigns **whole input files** to instances, so to use N instances **split the library into
≥ N CSV files** in the input prefix. You get **one `<file>.csv.out` per input file** — SageMaker does
**not** merge across files (`AssembleWith=Line` only reassembles the mini-batch chunks *within* one
file). Concatenate the `.out` files client-side for one combined result; a single-instance run needs
no split. See the [sample notebook](../aqpotency-sample.ipynb) for the shard → run → concat flow.

`GET /execution-parameters` advertises the container's batching preferences
(`MaxConcurrentTransforms=1`, `BatchStrategy=MULTI_RECORD`, `MaxPayloadInMB=6`).

See [`sample_data/input_batch_sample.csv`](sample_data/input_batch_sample.csv) and
[`sample_data/output_batch_sample.csv`](sample_data/output_batch_sample.csv).

### Billing

Billing is pro-rated (in seconds) at the machine-hour rate of the instance advertized in the marketplace listing.
