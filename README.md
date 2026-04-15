# Bring Your Own Model (BYOM) Template

## Purpose

This template defines a standardised, repeatable pattern for onboarding models into Databricks MLflow and deploying them to Model Serving.

It eliminates:
- One-off deployment scripts
- Manual model rewrites
- Broken lineage and unmanaged artifacts
- Inconsistent logging patterns across frameworks

It supports:
- Retraining existing models in Databricks with MLflow autolog
- Native MLflow flavours (XGBoost, scikit-learn, PyTorch) for externally trained artifacts
- Custom PyFunc models with preprocessing/postprocessing logic
- Unified deployment pattern regardless of framework

## Prerequisites

Minimum requirements:
- Databricks workspace with Unity Catalog enabled
- A catalog, schema, and volume for artifacts
- MLflow >= 3.0.0
- Python packages (installed via notebook `%pip` as needed): `xgboost`, `scikit-learn`, `torch`, `threadpoolctl>=3.5.0`

## Quickstart

### Path A — Already training in Databricks (no MLflow yet)

1. Create catalog, schema, and feature/label tables.
2. Run `01_retrain_with_mlflow/0_train_with_autolog.ipynb`.
3. Run evaluation, approval, and serving notebooks in `02_model_deployment/`.

### Path B — Externally trained model artifact

1. Create catalog, schema, volume.
2. Upload model artifacts to volume.
3. Run the appropriate logging notebook in `01_model_log_external/` (1a–1e).
4. Run evaluation, approval, and serving notebooks in `02_model_deployment/`.

Total time: ~30–60 minutes for standard frameworks.

## End-to-end flow

```
┌─────────────────────────────────────────────────────────┐
│  Entry point A                  Entry point B           │
│  01_retrain_with_mlflow/        01_model_log_external/  │
│  (train in Databricks)          (external artifact)     │
└────────────────────┬────────────────────┬───────────────┘
                     │                    │
                     └─────────┬──────────┘
                               ▼
                    02_model_deployment/
                    2_model_evaluation
                         ▼
                    3_model_approval
                         ▼
                    4a_model_serving  /  4b_batch_inference
```

## Repository structure

```
byo_model/
├── README.md
├── databricks.yml              # DABs bundle config
├── artifacts/
│   ├── 00_setup/               # Simulate external training (demo/test only — skip in real engagements)
│   ├── 01_retrain_with_mlflow/ # Entry point A: already training in Databricks, add MLflow
│   ├── 01_model_log_external/  # Entry point B: log and register externally trained artifact
│   └── 02_model_deployment/    # Evaluate → approve → serve (both paths converge here)
└── jobs/                       # DABs job definitions
```

## Artifacts directory

#### `artifacts/00_setup/`
Simulates external training. In real customer scenarios, **skip this** and use their existing artifacts.

| Notebook | Purpose |
|----------|---------|
| `0_export_model_and_artifacts.ipynb` | Trains sample models and exports artifacts into a UC volume. |

#### `artifacts/01_retrain_with_mlflow/`

For teams already training in Databricks without MLflow. Adds MLflow instrumentation to existing training code.

| Notebook | Purpose |
|----------|---------|
| `0_train_with_autolog.ipynb` | Adds `mlflow.autolog()` to existing training code, registers model to UC. |

#### `artifacts/01_model_log_external/`

For teams with externally trained model artifacts. Shared helpers are in `00_shared.ipynb`; run it from 1a–1e via `%run ./00_shared`.

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
- Registers model to UC

#### `artifacts/02_model_deployment/`

Handles evaluation, approval, and deployment. Both entry points converge here.

| Notebook | Purpose |
|----------|---------|
| `2_model_evaluation.ipynb` | Evaluate model on held-out data, log metrics |
| `3_model_approval.ipynb` | Approve/reject based on thresholds, assign Champion alias |
| `4a_model_serving.ipynb` | Deploy Champion to real-time serving endpoint, enable inference logging |
| `4b_batch_inference.ipynb` | Batch scoring |
| `_inference_reference.ipynb` | Inference cookbook |

## Artifact contract

Minimum required (Path B only):
- Serialized model artifact (framework-specific)

Strongly recommended:
- `canonical_input.json` for signature inference
- `checksums.json` for integrity validation

## Databricks Asset Bundles (DABs)

`databricks.yml` defines bundle variables, targets, and job wiring.

```bash
# Deploy to dev
databricks bundle deploy --target dev

# Run end-to-end pipeline
databricks bundle run e2e_pipeline_job --target dev
```

`jobs/` includes pre-defined jobs:
- `model-logging-job.yml` — log and register artifact
- `model-deployment-job.yml` — evaluate → approve → deploy
- `batch-inference-job.yml` — batch scoring
- `e2e-pipeline-job.yml` — full pipeline end-to-end

All jobs use serverless compute.

### Multi-model scale

This template is structured for a single model. For teams running multiple models (e.g. a suite of pricing or risk models with a shared pipeline structure), the recommended approach is to keep a single bundle and run jobs with per-model variable overrides:

```bash
databricks bundle run e2e_pipeline_job --var="model_name=pricing_model_a"
databricks bundle run e2e_pipeline_job --var="model_name=pricing_model_b"
```

Each model gets its own MLflow experiment, UC model version, and serving endpoint. Approval thresholds can also be overridden per model via `--var`. This works well when models share the same pipeline structure and differ only in training data or configuration.

If models diverge significantly in pipeline structure, consider a separate bundle per model.

### Promote Code

This template supports the Promote Code pattern. Point each target at a separate workspace and catalog:

```yaml
targets:
  dev:
    workspace:
      host: https://uat-ml.cloud.databricks.com
    variables:
      catalog_name: uat_catalog

  prod:
    workspace:
      host: https://prod-ml.cloud.databricks.com
    variables:
      catalog_name: prod_catalog
```

The same notebooks are deployed to both environments via `databricks bundle deploy`. Each environment trains against its own data and maintains its own model registry. No artifact crosses the workspace boundary.

Rollback within an environment is handled by reassigning the `Champion` alias to a previous model version — the serving endpoint resolves the alias at request time with no redeployment required. Cross-environment rollback is a `git revert` + re-run of `databricks bundle deploy`.

### CI/CD with GitHub Actions or Azure DevOps

For true CI/CD, wire `databricks bundle deploy` into your pipeline so that every merge to a branch triggers a deployment automatically — no manual CLI invocations required.

**GitHub Actions (example):**

```yaml
# .github/workflows/deploy.yml
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main
      - run: databricks bundle deploy --target prod
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
```

**Azure DevOps (example):**

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main]

steps:
  - script: |
      curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
      databricks bundle deploy --target prod
    env:
      DATABRICKS_HOST: $(DATABRICKS_HOST)
      DATABRICKS_TOKEN: $(DATABRICKS_TOKEN)
```

In both cases, store `DATABRICKS_HOST` and `DATABRICKS_TOKEN` (or a service principal OAuth credential) as secrets in the CI/CD system. The pipeline deploys notebooks and job definitions to the target workspace; the actual training and serving runs are then triggered separately via `databricks bundle run` or on schedule.

### Lakehouse Monitoring and Retraining

Once a model is serving, two additional layers are recommended to close the MLOps loop:

**Lakehouse Monitoring (data and model drift):**

Enable Lakehouse Monitoring on the inference log table written by Model Serving (configured in `4a_model_serving.ipynb`). This detects:
- Input feature drift (distribution shift in incoming requests)
- Prediction drift (output distribution changes over time)

Monitor the auto-generated `_drift_metrics` and `_profile_metrics` tables, or surface them in an AI/BI dashboard. Lakehouse Monitoring is configured in the Catalog UI or via the API — no additional code in this template is required.

**Automated retraining triggers:**

When drift exceeds a threshold, retraining can be triggered by:
1. A scheduled `databricks bundle run e2e_pipeline_job` (cron-based, simplest option)
2. A Lakeflow pipeline or DLT trigger that watches the drift metrics table and fires a job run when thresholds are breached
3. A CI/CD pipeline dispatch triggered by a monitoring alert webhook

The bundle variables (`accuracy_threshold`, `f1_threshold`) act as the gate in `3_model_approval.ipynb` — a retrain that does not improve on the current Champion will fail approval and not be promoted automatically.

## Design principles

- DataFrame in → DataFrame out
- Stable input/output contracts via MLflow model signatures
- Clear artifact boundaries
- Framework-agnostic deployment path
- Extendable via additional MLflow flavours
- Optional checksum validation for integrity

## What this template does not cover

The following should be layered on top depending on customer maturity:

- Automated retraining triggers
- Blue/green or canary deployments
- Multi-model routing
- CI/CD pipeline (GitHub Actions / Azure DevOps triggering bundle deploy)
- Dev/prod workspace separation
- Data validation before training

## Troubleshooting

- **Volume not found** — Check catalog, schema, and volume names in widgets.
- **Import / module errors** — Run `%pip install` in the notebook and restart Python.
- **Registration fails** — Check Unity Catalog permissions.
- **Deployment fails** — Ensure the model has the Champion alias and approval tag set by `3_model_approval`.
- **DABs deploy fails** — Ensure `DATABRICKS_HOST` is set or hardcoded in the dev target. `workspace.host` does not support variable interpolation.
