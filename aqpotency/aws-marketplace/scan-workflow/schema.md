# Scan Workflow — Input / Output Schema

This document describes every field accepted and returned by the **scan mode** of the **AQPotency**
model package. Given a **ligand + a curated target panel**, the model scores the ligand against
every target in the panel and returns the hits ranked by predicted potency, with off-target alerts.

The model exposes **one synchronous endpoint**, `POST /invocations`, plus a `GET /ping` health
check. Request and response bodies are both `application/json`. Every response has the shape
`{"predictions": [...], "warnings": [...]}` and carries the `X-Amzn-Inference-Metering` header.

---

## Request

```json
{
  "mode": "scan",
  "applicability_domain": true,
  "instances": [
    {
      "smiles": "COc1cc2ncnc(Nc3ccc(F)c(Cl)c3)c2cc1OCCCN1CCOCC1",
      "panel": "human_kinome",
      "threshold": 7.0,
      "top_n": 20
    }
  ]
}
```

> Each instance scans **one ligand** against one named `panel`. Scan options (`panel`, `threshold`,
> `top_n`) are set **per instance**.

### Top-level fields

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `mode` | string | **Required** | Set to `"scan"`. `mode` is required — a missing or unrecognized value returns `422` (there is no default). |
| `applicability_domain` | bool | Optional | Toggle the applicability-domain computation. Default `true`; set `false` to skip AD for a faster response (per-hit `applicability_domain` returns `null`). Does **not** change billing. |
| `instances` | array&lt;object&gt; | **Required** | One or more scan requests, each one ligand vs one panel. At most **3** instances (`SCAN_MAX_LIGANDS`). |

### Instance object (`ScanInstance`)

| Name | Type | Required | Description | Example |
|------|------|----------|-------------|---------|
| `smiles` | string | **Required** | Ligand structure as a **SMILES** string. Canonicalized (RDKit) server-side; unparseable → `422`. | `"CCO"` |
| `panel` | string | Optional | Named target panel to scan against. One of `bowes_safety_panel` (46 targets), `human_kinome` (604), `proteome` (20,431). Default `bowes_safety_panel`. Unknown panel → `422`. | `"human_kinome"` |
| `threshold` | float | Optional | pIC50 threshold that drives `prob_above_threshold` and `alert`. Default `7.0`. | `7.0` |
| `top_n` | int | Optional | Return only the top-N targets by predicted pIC50. Omit to return all. Bounds the **response** size (all panel targets are still scored and billed). | `20` |

---

## Response

```json
{
  "predictions": [
    {
      "smiles": "COc1cc2ncnc(Nc3ccc(F)c(Cl)c3)c2cc1OCCCN1CCOCC1",
      "panel": "human_kinome",
      "n_targets": 604,
      "hits": [
        {
          "uniprot_id": "P00533",
          "target_name": "EGFR",
          "predicted_mean": 7.64,
          "predicted_sigma": 1.12,
          "prob_above_threshold": 0.90,
          "alert": true,
          "applicability_domain": 1.0,
          "gene_symbol": "EGFR",
          "target_class": "Kinase",
          "source": "curated"
        }
      ]
    }
  ],
  "warnings": []
}
```

One element in `predictions` per input ligand — a `ScanResult`:

| Name | Type | Description | Example |
|------|------|-------------|---------|
| `smiles` | string | The scanned ligand, echoing the request. | `"CCO"` |
| `panel` | string | The panel that was scanned, echoing the request. | `"human_kinome"` |
| `n_targets` | int | Number of panel targets scored (also the billed unit for this ligand). | `604` |
| `hits` | array&lt;object&gt; | Per-target predictions, ranked by `predicted_mean` (desc). Trimmed to `top_n` when supplied. | see below |

Each element of `hits` is a `ScanHit`:

| Name | Type | Description | Example |
|------|------|-------------|---------|
| `uniprot_id` | string | Target UniProt accession. | `"P00533"` |
| `target_name` | string \| null | Resolved protein name, or `null`. | `"EGFR"` |
| `predicted_mean` | float | Predicted potency as **pIC50** (higher = more potent). | `7.64` |
| `predicted_sigma` | float | Uncertainty (standard deviation) on `predicted_mean`. | `1.12` |
| `prob_above_threshold` | float | P(pIC50 &gt; `threshold`), computed from the mean/sigma (~0–1). | `0.90` |
| `alert` | bool | `true` when `prob_above_threshold` &gt; 0.5 — a **likely off-target hit**. | `true` |
| `applicability_domain` | float | In-/out-of-domain score (~0–1) for this ligand–target prediction. | `1.0` |
| `gene_symbol` | string \| null | Optional annotation — target gene symbol. | `"EGFR"` |
| `target_class` | string \| null | Optional annotation — target class (e.g. `"Kinase"`). | `"Kinase"` |
| `source` | string \| null | Optional annotation — provenance of the panel entry. | `"curated"` |

> **Alerts and applicability domain are screening signals.** An `alert: true` hit is a *likely*
> off-target interaction, not a confirmed one — weigh it with `prob_above_threshold` and
> `applicability_domain`. A ligand whose applicability domain is low (below `AD_WARN_THRESHOLD`,
> default `0.4`) also adds a note to the response `warnings[]`.

---

## Errors

Failures return a non-2xx status with an error body:

```json
{ "error": "<message>" }
```

| Code | Meaning |
|------|---------|
| `422` | Schema or guardrail violation — missing `smiles`, an **unparseable SMILES**, an **unknown `panel`**, more than 3 ligands, or an otherwise malformed payload. |
| `500` | Unexpected runtime error during inference. |

> The container does not emit `400`. A real-time invocation may still receive a SageMaker
> **endpoint-level** `504` if the request exceeds the ~60 s response limit. `GET /ping` returns
> `503` until the model is loaded, `200` after.

---

## Guardrails

Enforced **before** inference; violations are rejected (not clamped) with `422`:

- **Known mode / known panel.** `mode` is **required** and must be one of `potency` / `scan` /
  `screen` / `applicability_domain` / `selectivity`; `panel` must be one of `bowes_safety_panel` /
  `human_kinome` / `proteome`.
- **Parseable SMILES.** Every `smiles` must parse (RDKit); unparseable → `422`.

**Applicability-domain warning (not a rejection):** a ligand whose `applicability_domain` is below
`AD_WARN_THRESHOLD` (default `0.4`) still returns hits, but adds a note to `warnings[]`.

**Response size.** ⚠️ A full `proteome` scan returns ~20,431 hits — a body of **~5.8 MB**, near the
**6 MB** real-time payload cap. Use `top_n` to bound the response. A request body is **≤ 6 MB** and a
**real-time** invocation must respond within **~60 s** (`proteome` ~24 s).

---

## Batch transform

> ⚠️ **Not recommended.** SandboxAQ does not actively support batch transform jobs for interactive
> use, and recommend the batching available via real-time inference. (Batch transform is used
> internally only for model-package validation.)

The schema is identical for batch transform. Each input file is **one complete request body**
(`{ "mode": "scan", "instances": [...] }`, ≤ 3 ligands). The model processes one file per request
(`BatchStrategy=SingleRecord`, `SplitType=None`, `ContentType=application/json`, ≤ 6 MB), writing one
`*.out` file containing the response JSON. See
[`sample_data/input_realtime_sample.json`](sample_data/input_realtime_sample.json) and
[`sample_data/output_realtime_sample.json`](sample_data/output_realtime_sample.json). Batch transform
runs on **`ml.m5.2xlarge`** (see the [hardware selection guide](../hardware_selection_guide.md)).

---

## Billing

Successful (2xx) responses carry a metering header for AWS Marketplace:

```
X-Amzn-Inference-Metering: {"Dimension":"inference.count","ConsumedUnits":N}
```

`ConsumedUnits` = the **total number of panel targets scored**, summed over the ligands in the
request (= Σ `n_targets`) — ≈ 46 per `bowes_safety_panel` ligand, 604 per `human_kinome`, 20,431 per
`proteome`. `top_n` bounds the response, **not** the bill. Failed (4xx/5xx) responses are **not**
billed.

See more in the [billing guide](../billing_guide.md).
