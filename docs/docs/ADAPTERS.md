# Reusable adapter library

Common datasource, data-engine, DAG-runtime, and orchestration adapters are
modules in the `transformation_dsl` distribution. A consuming project installs
this library and selects adapters through an execution profile; it does not need
to create or publish one package per common integration.

## Adapter families

| Family | Port | Built-in scaffold modules |
|---|---|---|
| DAG runtime | `DagRuntimeAdapter` | `hamilton`, `native` |
| Orchestration | `OrchestrationBackend` | `airflow`, `in_process` |
| Data engine | `EngineAdapter` | `pandas`, `polars`, `duckdb`, `spark` |
| Datasource | `DatasourceAdapter` | `snowflake`, `databricks`, `oracle`, `blob` |

The built-in catalog is registered directly by
`transformation_dsl.adapters.catalog.register_builtin_adapters`. The generic
`AdapterRegistry` is also reusable by another service, CLI, notebook, or worker.

## One distribution, optional dependencies

Vendor modules must not import heavy SDKs at module import time. A future concrete
adapter should load its driver when it is configured or first used and return a
clear missing-extra error when the dependency is absent.

Consumers select only the extras needed by their worker image:

```bash
uv add "transformation-dsl-platform[hamilton,snowflake,spark]"
```

This keeps a single shared implementation while allowing small API and worker
images. API/control-plane images do not need every data driver installed.

## Execution profiles

Workflow YAML stays portable:

```yaml
dsl_version: "1.0"
id: ledger-flow
execution_profile: standard
steps:
  - id: transform
    uses: smpl.lib.ledger.transform
    engine: spark
```

Deployment configuration owns framework selection:

```json
{
  "standard": {
    "orchestrator": "airflow",
    "dag_runtime": "hamilton",
    "dag_runtime_options": {
      "executor": "default"
    }
  }
}
```

Changing `dag_runtime` to `native` or a future adapter does not modify workflow
definitions. Framework-specific options remain inside
`dag_runtime_options` and never enter the canonical plan semantics.

## Optional external extensions

Entry-point discovery remains available for organization-specific frameworks or
private systems that do not belong in the common library:

| Entry-point group | Contract |
|---|---|
| `transformation_dsl.dag_runtimes` | `DagRuntimeAdapter` |
| `transformation_dsl.orchestrators` | `OrchestrationBackend` |
| `transformation_dsl.engines` | `EngineAdapter` |
| `transformation_dsl.datasources` | `DatasourceAdapter` |

External packages are an escape hatch, not the default architecture.

## Conformance requirements

Every DAG runtime must pass the same contract suite for:

- dependency ordering;
- reference binding;
- conditional skipping;
- pre-validation failures;
- output propagation;
- retry and timeout interpretation;
- error normalization and cancellation;
- deterministic plan fingerprints.

An adapter that cannot preserve a plan feature must reject it during capability
validation rather than silently changing behavior.
