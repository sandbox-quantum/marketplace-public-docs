# Getting Started

Below are details on how to quickly get started with the [sample notebook](./aqcat-sample.ipynb) in this directory. You can find more robust examples in the [MLP model docs](../mlp-model/) and [min energy adsorption docs](../adsorption-workflow/) directories.

All examples are for using SageMaker's real-time inference capabilities, which is the main use case supported.

## Prerequisites

- An **AWS account** that can subscribe to the [AQCat listing](https://aws.amazon.com/marketplace/pp/prodview-smq2zrd5oqpnc).
- A **SageMaker execution role** with permission to create models/endpoints/transform jobs. In SageMaker Studio this is detected
  automatically (`sagemaker.get_execution_role()`); otherwise supply your own role ARN.
- **Quota for at least one of the [allowed instance types](./hardware_selection_guide.md)** in your region (the sample notebook deploys `ml.g5.xlarge`).

## Quickstart: Running the sample notebook

1. Subscribe to the [AQCat listing](https://aws.amazon.com/marketplace/pp/prodview-smq2zrd5oqpnc) in the AWS web console. Once you're at the listing page, click `View purchase options` > then `Subscribe` on the next page.

### Option A — Amazon SageMaker Studio web UI (fastest)

2. Launch [SageMaker Studio](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-updated-launch.html).
3. Clone this repo (or upload [the notebook](./aqcat-sample.ipynb)) into Studio.
4. Open the notebook and run the cells top to bottom.

### Option B — local / other environment
2. Clone this repo into your environment and navigate to this directory.
3. Ensure you have the packages specified in [requirements.txt](./requirements.txt) installed in your environment. Otherwise, you can run:
```bash
pip install -r requirements.txt
aws configure          # credentials + default region
```
4. Open the notebook and run the cells top to bottom.

## Cost & cleanup
> 💰 The notebook creates **billable** resources. **Run section 5 (Clean up)** when you finish to avoid leaving a forgotten endpoint running and billing continuously. Billing is metered per successful response via `X-Amzn-Inference-Metering`, with every call of the MLP resulting in one inference metered (whether explicitly called in `mlp` mode or called multiple times in `min-adsorption-energy-workflow` mode).

## Troubleshooting

| Problem | Fix |
|---|---|
| `ValueError: product is not available in <region>` | The product isn't published in your session Region yet. Run in a listed Region, or if you know that the model exists in your region, add the appropriate ARN to `model_package_map`. Email `support.aisim (at) sandboxaq.com` if you would like for support to be added to your region. |
| `AccessDenied` | The execution role needs SageMaker + S3 permissions (and the account must be subscribed to the listing). |
| Adsorption-style error on an MLP call | Make sure the request body includes `"mode": "mlp"`. |
| `ResourceLimitExceeded` on deploy | Request a quota increase for `ml.g5.xlarge` (what the notebook deploys), or use another [allowed instance](./hardware_selection_guide.md) such as `ml.g6.xlarge`. |

## Need support?
Email `support.aisim (at) sandboxaq.com` for any bug reports, feature requests, or general support questions.
