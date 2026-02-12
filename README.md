# Bring Your Own Model (BYOM) Template

## Purpose

This template defines a standardised, repeatable pattern for onboarding externally trained models into Databricks.

This template eliminates:
- One-off deployment scripts
- Manual model rewrites
- Broken lineage and unmanaged artifacts
- Inconsistent logging patterns across frameworks

It supports:
- Native MLflow flavours (XGBoost, scikit-learn, PyTorch)
- Custom PyFunc models with preprocessing/postprocessing logic
- Unified deployment pattern regardless of framework

## Prerequisites

Minimum requirements:
- Databricks workspace with Unity Catalog enabled
- A catalog, schema, and volume for artifacts
- MLflow >= 3.0.0
- Python packages (installed via notebook `%pip` as needed): `xgboost`, `scikit-learn`, `torch`, `threadpoolctl>=3.5.0`

## Quickstart
1. Create catalog, schema, volume.
2. Upload model artifacts to volume.
3. Run appropriate logging notebook (1a-1e).
4. Run evaluation notebook.
5. Run model approval notebook.
6. Run serving or batch inference notebook.

Total time: ~30-60 minutes for standard frameworks.

## End-to-end flow

The workflow follows three stages:

1. **Stage 1 - Artifact Export** — What does the customer already have? 
    - E.g. `.pkl`, `.pt`, `.json`, preprocessing configs.
2. **Stage 2 - Log & register** — How do we make it governable, traceable and production-ready? 
    - This stage loads artifacts from UC volume -> validates checksums (optional but recommended) -> infers signature -> logs to MLflow -> registers to UC -> assigns a Champion alias.
3. **Stage 3 - Deploy** — How do we operationalise it?
    - Options: real-time model serving endpoint or batch inference.
    - Once registered, the same deployment code works for all model types.

## Operational Considerations

This template covers onboarding and deployment only.

Not included:
- Drift detection
- Automated retraining
- Blue/green serving
- Canary deployments
- Multi-model routing

These should be layered on top depending on customer maturity.

## Initial setup

### Create Unity Catalog resources

```sql
CREATE CATALOG IF NOT EXISTS <catalog>;
CREATE SCHEMA IF NOT EXISTS <catalog>.<schema>;
CREATE VOLUME IF NOT EXISTS <catalog>.<schema>.<volume>;
```

Notebooks use widget defaults: `catalog_name`, `schema_name`, `volume_name`. Ensure these are aligned before running.

## Artifact contract

Minimum required:
- Serialized model artifact (framework-specific)

Strongly recommended:
- `canonical_input.json` for signature inference
- `checksums.json` for integrity validation

## Repository structure

```
byo_model/
├── README.md
├── databricks.yml              # Optional: DABS bundle config
├── artifacts/                  # All notebooks live here
│   ├── 00_setup/
│   ├── 01_model_log/
│   └── 02_model_deployment/
└── jobs/                       # Optional: DABS job definitions
```

## Artifacts directory

#### `artifacts/00_setup/`
Simulates external training. In real customer scenarios, **skip this** and use their existing artifacts.

| Notebook | Purpose |
|----------|---------|
| `0_export_model_and_artifacts.ipynb` | Trains sample models and exports native + PyFunc artifacts into a UC volume. |

#### `artifacts/01_model_log/`

Logs and registers externally trained models. Shared helpers are in `00_shared.ipynb`; run it from 1a–1e via `%run ./00_shared`.

| Notebook | Model type | Logging |
|----------|------------|---------|
| `1a_native_xgb.ipynb` | XGBoost | `mlflow.xgboost` |
| `1b_native_sklearn.ipynb` | scikit-learn | `mlflow.sklearn` |
| `1c_pyfunc_xgb.ipynb` | PyFunc XGB | `mlflow.pyfunc` |
| `1d_dl_pyfunc.ipynb` | DL PyFunc | `mlflow.pyfunc` |
| `1e_native_torch.ipynb` | PyTorch | `mlflow.pytorch` |

Each notebook:
- Loads artifacts from UC volume
- Validates checksums (if present)
- Infers signature
- Logs to MLflow
- Registers model
- Assigns Champion alias

#### `artifacts/02_model_deployment/`

Handles evaluation, approval and deployment.

| Notebook | Purpose |
|----------|---------|
| `2_model_evaluation.ipynb` | Evaluate model |
| `3_model_approval.ipynb` | Approve/reject based on thresholds |
| `4a_model_serving.ipynb` | Create model serving endpoint for real-time inference |
| `4b_batch_inference.ipynb` | Batch scoring |
| `_inference_reference.ipynb` | Inference cookbook.

Switch models by updating `model_name`.

## Using with customer artifacts

Typical flow:

1. Upload model artifacts to a UC volume.
2. Optionally include `checksums.json` to the volume for integrity validation.
3. Run the appropriate model logging notebook (1a-1e).
4. Run evaluation and approval.
5. Deploy model serving endpoint for real-time inference or run batch inference.

## Databricks Asset Bundles (optional)

This repo includes optional DABs configuration.

`databricks.yml`

Defines:
- Bundle name
- Variables (catalog, schema, volume, model name, thresholds)
- Targets (dev/prod)
- Job wiring
- Notebook sync rules

You can:
- Modify variables
- Swap notebook paths
- Integrate into your own deployment structure

`jobs/`

Includes pre-defined jobs:
- `model-logging-job.yml`
- `model-deployment-job.yml`
- `batch-inference-job.yml`
- `e2e-pipeline-job.yml`

These are example. You may replace with customer-specific jobs.

## Design principles

- DataFrame in → DataFrame out
- Stable input/output contracts
- Clear artifact boundaries
- Framework-agnostic deployment path
- Extendable via additional MLflow flavours
- Optional checksum validation for integrity

## Troubleshooting

- **Volume not found** — Check catalog, schema, and volume names.
- **Import / module errors** — Run `%pip install` in the notebook and restart Python.
- **Registration fails** — Check Unity Catalog permissions and (if used) checksum keys.
- **Deployment fails** — Ensure the model is approved and has the Champion alias.