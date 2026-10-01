# Selectivity Mode — Input / Output Schema

> ⚠️ **Provisional.** The selectivity metric is being finalized. Raw on-target and off-target potency
> values are returned so any selectivity metric can be derived client-side; the `selectivity`
> comparison field may change.

The **selectivity mode** of the **AQPotency** model package compares a ligand's potency at an
on-target vs a set of off-targets.

Endpoint `POST /invocations` (+ `GET /ping`). Request/response are `application/json`; every response
is `{"predictions": [...], "warnings": [...]}` with the `X-Amzn-Inference-Metering` header. `mode` is
**required** — a missing/unknown value returns `422`.

---

## Request

```json
{
  "mode": "selectivity",
  "instances": [
    {
      "smiles": "COc1cc2ncnc(Nc3ccc(F)c(Cl)c3)c2cc1OCCCN1CCOCC1",
      "uniprot_id": "P00533",
      "off_targets": ["P07550", "P11362"]
    }
  ]
}
```

### Top-level fields

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `mode` | string | **Required** | Set to `"selectivity"`. |
| `instances` | array&lt;object&gt; | **Required** | One or more ligand/on-target/off-targets groups. |

### Instance object

| Name | Type | Required | Description | Example |
|------|------|----------|-------------|---------|
| `smiles` | string | **Required** | Ligand SMILES. Canonicalized (RDKit) server-side. | `"CCO"` |
| `uniprot_id` | string | **Required** | **On-target** UniProt accession; must be in the vocabulary. | `"P00533"` |
| `off_targets` | string[] | **Required** | UniProt accessions to compare against the on-target; each must be in the vocabulary. | `["P07550", "P11362"]` |

---

## Response

```json
{
  "predictions": [
    {
      "smiles": "COc1cc2ncnc(Nc3ccc(F)c(Cl)c3)c2cc1OCCCN1CCOCC1",
      "uniprot_id": "P00533",
      "target_name": "EGFR",
      "potency_mean": 7.64,
      "potency_sigma": 1.12,
      "applicability_domain": 0.83,
      "off_targets": [
        { "uniprot_id": "P07550", "target_name": "ADRB2", "potency_mean": 5.10, "potency_sigma": 1.20, "applicability_domain": 0.29, "selectivity": 2.54 },
        { "uniprot_id": "P11362", "target_name": "FGFR1", "potency_mean": 6.80, "potency_sigma": 0.95, "applicability_domain": 0.61, "selectivity": 0.84 }
      ]
    }
  ],
  "warnings": []
}
```

One element per ligand. The on-target fields (`potency_mean` / `potency_sigma` /
`applicability_domain` / `target_name`) are the potency at `uniprot_id`. Each `off_targets[]` entry
carries that off-target's own potency, AD, and a **`selectivity`** comparison vs the on-target.

| Field (per off-target) | Type | Description |
|------|------|-------------|
| `uniprot_id` / `target_name` | string / string\|null | The off-target. |
| `potency_mean` / `potency_sigma` | float | Off-target potency (pIC50) and uncertainty. |
| `applicability_domain` | float | Off-target AD score. |
| `selectivity` | float | **Provisional** on-target vs off-target comparison (e.g. on-target − off-target pIC50; higher = more selective for the on-target). Derive your own metric from the raw values if preferred. |

---

## Errors

```json
{ "error": "<message>" }
```

| Code | Meaning |
|------|---------|
| `422` | Missing `smiles`/`uniprot_id`/`off_targets`, an unknown on- or off-target accession, unparseable SMILES, or more than `MAX_SELECTIVITY` (**15**) total on+off evaluations. |
| `500` | Unexpected runtime error. |

---

## Guardrails

- **Known mode.** `mode` required and valid.
- **Targets in vocabulary.** On-target `uniprot_id` and every `off_targets` accession must be known.
- **Pair count.** ≤ `MAX_SELECTIVITY` (**15**) total on+off evaluations per request.

---

## Billing

```
X-Amzn-Inference-Metering: {"Dimension":"inference.count","ConsumedUnits":N}
```

`ConsumedUnits` = **on-target + off-target evaluations** per ligand (1 + N off-targets). Failed
(4xx/5xx) responses are **not** billed. See the [billing guide](../billing_guide.md).
