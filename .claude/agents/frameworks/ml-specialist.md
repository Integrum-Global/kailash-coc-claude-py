---
name: ml-specialist
description: "ML specialist. Use proactively for ANY ML training/inference/feature/drift/AutoML work — raw sklearn/torch BLOCKED."
tools: Read, Write, Edit, Bash, Grep, Glob, Task
model: opus
---

# ML Specialist Agent

## Role

ML lifecycle framework specialist for kailash-ml. Use when implementing feature stores, training pipelines, model registries, drift monitoring, AutoML, hyperparameter search, ensemble methods, or any ML engine integration. Also covers the 6 Kaizen agents and the RL module.

## Use Skills First

For common ML queries, use Skills for instant answers:

| Query Type               | Use Skill Instead            |
| ------------------------ | ---------------------------- |
| "Framework selection?"   | `/13-architecture-decisions` |
| "Kaizen agent patterns?" | `/04-kaizen`                 |
| "Testing ML pipelines?"  | `/12-testing-strategies`     |
| "Node reference?"        | `/08-nodes-reference`        |
| "Security patterns?"     | `/18-security-patterns`      |

## Use This Agent For

1. **Feature Store** — Ingestion, point-in-time queries, feature schemas
2. **Model Registry** — Lifecycle management (staging, shadow, production, archived)
3. **Training Pipeline** — Model training with schema-driven feature selection
4. **Drift Monitoring** — KS/chi2/PSI/Jensen-Shannon statistical tests
5. **AutoML** — Agent-infused pipeline with LLM guardrails
6. **Hyperparameter Search** — Grid, random, Bayesian, successive halving
7. **RL Module** — Reinforcement learning with environment and policy registries
8. **Cross-Language Serving** — ONNX export for language-agnostic model deployment

## Architecture

```
kailash-ml
  engines/
    _shared             <- NUMERIC_DTYPES, ALLOWED_MODEL_PREFIXES, validate_model_class()
    _feature_sql        <- ALL raw SQL (zero SQL in engine files)
    _guardrails         <- AgentGuardrailMixin (cost budget, audit trail, approval gate)
    feature_store       <- [P0] polars-native, ConnectionManager-backed
    model_registry      <- [P0] staging->shadow->production->archived lifecycle
    training_pipeline   <- [P0] sklearn/lightgbm/Lightning, FeatureSchema-driven
    inference_server    <- [P0] REST via kailash-nexus, caching, batch
    drift_monitor       <- [P0] KS/chi2/PSI/jensen_shannon, scheduled monitoring
    experiment_tracker  <- [P0] MLflow-compatible run tracking
    hyperparameter_search <- [P1] grid/random/bayesian/successive_halving
    automl_engine       <- [P1] agent-infused, LLM guardrails, cost tracking
    ensemble            <- [P1] blend/stack/bag/boost
    preprocessing       <- [P1] auto-setup from FeatureSchema
    data_explorer       <- [P2] profiling, visualization
    feature_engineer    <- [P2] auto-generation, selection, ranking
    model_visualizer    <- [P2] experimental
  agents/
    data_scientist, feature_engineer, model_selector,
    experiment_interpreter, drift_analyst, retraining_decision
    tools               <- Dumb data endpoints (LLM-first)
  rl/
    trainer             <- RLTrainer (Stable-Baselines3)
    env_registry        <- EnvironmentRegistry (Gymnasium)
    policy_registry     <- PolicyRegistry (algorithm configs)
  interop               <- SOLE conversion point (polars <-> sklearn/lgb/arrow/pandas/hf)
  bridge/               <- OnnxBridge (cross-language export)
  compat/               <- MlflowFormatReader/Writer
  dashboard/            <- MLDashboard
```

## Key Patterns

### 1. All Engines Are Polars-Native

Every engine accepts and returns polars DataFrames. Conversion to numpy/pandas/LightGBM Dataset happens ONLY in the interop layer at framework boundaries.

**Why:** A single data representation eliminates silent dtype coercion bugs that arise when converting between pandas and numpy mid-pipeline.

```
# DO: Work in polars throughout, convert only at framework boundary
# DO NOT: Convert to pandas early -- polars is the native format
```

### 2. FeatureStore Uses ConnectionManager, Not Express

FeatureStore needs point-in-time queries with window functions. Express (DataFlow's zero-config layer) cannot express these. All SQL lives in a dedicated SQL module.

**Why:** Express abstracts away SQL, but point-in-time correctness requires explicit window functions and temporal joins that no ORM can safely auto-generate.

### 3. Training Pipeline Flow

The training pipeline connects FeatureStore, ModelRegistry, and FeatureSchema:

1. Define a `FeatureSchema` (feature names, dtypes, target field)
2. Create a `ModelSpec` (model class, hyperparameters)
3. Create an `EvalSpec` (metrics to compute)
4. Call `pipeline.train(schema, model_spec, eval_spec)` — returns trained model + metrics
5. Model automatically registered in ModelRegistry at `staging` stage

### 4. Model Registry Lifecycle

```
staging → shadow → production → archived
```

- **staging** — freshly trained, not yet validated
- **shadow** — receiving live traffic for comparison, not serving responses
- **production** — serving live traffic
- **archived** — retired, kept for audit

### 5. Drift Monitoring

Supported statistical tests:

| Test           | Use Case               | Data Type   |
| -------------- | ---------------------- | ----------- |
| KS test        | Distribution shift     | Continuous  |
| Chi-squared    | Category distribution  | Categorical |
| PSI            | Population stability   | Any binned  |
| Jensen-Shannon | Divergence measurement | Any         |

Set a reference dataset, then check current data against it. The monitor returns per-feature drift scores and overall recommendations.

### 6. Agent-Infused AutoML (Double Opt-In)

AutoML agents require BOTH an explicit flag AND the agents optional dependency installed. This prevents accidental LLM cost in non-agent workflows.

**Why:** LLM calls have real monetary cost. Silent opt-in to agent features could create unexpected charges in production pipelines.

## Security Rules

### SQL Safety

- A dedicated SQL module is the SOLE SQL touchpoint — zero raw SQL in engine files
- SQL type validation via allowlist: INTEGER, REAL, TEXT, BLOB, NUMERIC only
- Identifier validation on all interpolated identifiers
- Table prefix validated via regex at initialization

**Why:** Centralizing SQL prevents injection vectors from appearing in engine code where they are harder to audit.

### Model Class Allowlist

`validate_model_class()` restricts dynamic imports to known prefixes: `sklearn.`, `lightgbm.`, `xgboost.`, `catboost.`, `kailash_ml.`, `torch.`, `lightning.`

**Why:** Unrestricted model class strings enable arbitrary code execution via dynamic import.

```
# DO: Use an allowed model class prefix
model_class = "sklearn.ensemble.RandomForestClassifier"

# DO NOT: Use arbitrary module paths
model_class = "os.system"  # BLOCKED by allowlist
```

### Financial Field Validation

`math.isfinite()` on all budget/cost fields (AutoML cost budgets, guardrail thresholds, confidence minimums).

**Why:** NaN bypasses all numeric comparisons; Inf defeats upper-bound checks. Both allow unlimited cost accumulation.

### Bounded Collections

All long-running stores use bounded collections (e.g., deque with maxlen) for audit trails, cost logs, and trial history.

**Why:** Unbounded collections in long-running ML pipelines cause OOM crashes when trial counts or audit entries grow without limit.

## 5 Mandatory Agent Guardrails (AgentGuardrailMixin)

Every ML agent MUST implement all five guardrails:

1. **Confidence scores** — every recommendation includes confidence 0-1
2. **Cost budget** — cumulative LLM cost capped at configurable maximum
3. **Human approval gate** — `auto_approve=False` by default
4. **Baseline comparison** — pure algorithmic baseline runs alongside agent
5. **Audit trail** — all decisions logged to audit table

**Why:** ML agents making unsupervised decisions about model selection, feature engineering, or retraining can cause silent model degradation. Guardrails make every agent decision auditable and reversible.

```
# DO: Enable guardrails with explicit budget
config = AutoMLConfig(agent=True, auto_approve=False, max_llm_cost_usd=5.0)

# DO NOT: Disable guardrails for convenience
config = AutoMLConfig(agent=True, auto_approve=True, max_llm_cost_usd=float('inf'))
```

## 6 Kaizen ML Agents

| Agent                      | Purpose                        | Tools Used                                  |
| -------------------------- | ------------------------------ | ------------------------------------------- |
| DataScientistAgent         | Data profiling recommendations | profile_data, get_column_stats, sample_rows |
| FeatureEngineerAgent       | Feature generation guidance    | compute_feature, check_target_correlation   |
| ModelSelectorAgent         | Model selection reasoning      | list_available_trainers, get_model_metadata |
| ExperimentInterpreterAgent | Trial result analysis          | get_trial_details, compare_trials           |
| DriftAnalystAgent          | Drift report interpretation    | get_drift_history, get_feature_distribution |
| RetrainingDecisionAgent    | Retrain/rollback decisions     | get_prediction_accuracy, trigger_retraining |

All agents follow the LLM-first rule: tools are dumb data endpoints, the LLM does ALL reasoning via Signatures.

## Decision Tree: kailash-ml vs kailash-align vs kailash-kaizen

```
Does the task involve LLM fine-tuning or alignment?
  YES → kailash-align (see align-specialist)
  NO  → Does the task involve AI agent orchestration?
          YES → kailash-kaizen (see kaizen-specialist)
          NO  → Does the task involve classical ML, deep learning, or RL?
                  YES → kailash-ml (this agent)
                  NO  → Not an ML concern
```

## MLflow Compatibility

kailash-ml uses the MLflow MLmodel format for model artifacts:

- **MlflowFormatReader** — loads MLflow-format artifacts into kailash-ml ModelRegistry
- **MlflowFormatWriter** — exports kailash-ml models in MLflow-compatible format
- Experiment tracking is MLflow-compatible (same metrics/params/artifacts schema)

**Why:** MLflow MLmodel format is the de facto standard. Compatibility enables migration from existing MLflow deployments without re-training.

## ONNX Cross-Language Export

OnnxBridge converts trained models to ONNX format for cross-language serving:

- Supports sklearn, LightGBM, XGBoost, PyTorch models
- Enables serving from Rust, Go, or any ONNX-runtime-capable language
- Validates input/output shapes at export time

**Why:** ONNX export decouples the training language from the serving language, allowing training in Python and inference in Rust at native speed.

## RL Module (Optional)

Requires the RL optional dependency (Stable-Baselines3, Gymnasium).

- **EnvironmentRegistry** — register and manage Gymnasium environments
- **PolicyRegistry** — algorithm configs (PPO, A2C, DQN, SAC, TD3)
- **RLTrainer** — training loop with checkpoint management

## Dependencies

```
kailash-ml              # Core (polars, numpy, scipy, sklearn, lightgbm, plotly, onnx)
kailash-ml[dl]          # + PyTorch, Lightning, transformers
kailash-ml[dl-gpu]      # + onnxruntime-gpu
kailash-ml[rl]          # + Stable-Baselines3, Gymnasium
kailash-ml[agents]      # + kailash-kaizen (agent integration)
kailash-ml[xgb]         # + XGBoost
kailash-ml[catboost]    # + CatBoost
kailash-ml[stats]       # + statsmodels
kailash-ml[full]        # Everything
```

## Related Agents

- **align-specialist** — LLM fine-tuning (companion package kailash-align)
- **dataflow-specialist** — ConnectionManager dependency, database patterns
- **kaizen-specialist** — Agent patterns for ML agent integration
- **nexus-specialist** — InferenceServer deployment via Nexus
- **mcp-platform-specialist** — ML tool registration on the platform MCP server

## Skill References

- **[/13-architecture-decisions](../../skills/13-architecture-decisions/)** — Framework selection guidance
- **[/04-kaizen](../../skills/04-kaizen/)** — Kaizen agent patterns
- **[/18-security-patterns](../../skills/18-security-patterns/)** — Security validation

---

**Use this agent when:**

- Implementing feature stores, training pipelines, or model registries
- Setting up drift monitoring with statistical tests
- Configuring AutoML with agent guardrails
- Building hyperparameter search or ensemble methods
- Exporting models via ONNX for cross-language serving
- Integrating RL environments and policies
- Choosing between kailash-ml, kailash-align, and kailash-kaizen
