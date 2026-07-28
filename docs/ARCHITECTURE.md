# Architecture

## Goal

This repository is an independent transformation platform. Other services submit
versioned YAML workflows through FastAPI. The platform validates and compiles a
workflow into an engine-neutral execution plan, then delegates durable scheduling
to Airflow. Datasource and data-engine integrations are replaceable plug-ins.

This milestone is a scaffold: the package boundaries, domain contracts, extension
points, API shape, container, and tests exist; parsing, compilation, persistence,
Airflow submission, and vendor operations intentionally do not.

## Architecture choice

Start as a modular monolith with separately deployable control-plane and runner
processes. This preserves strong module boundaries without creating an operational
network hop for every class. Split a module into a service only when scale,
security, ownership, or release cadence requires it.

The two runtime planes are:

- **Control plane** — FastAPI, workflow catalog, DSL parser/compiler, plug-in
  catalog, run metadata, and Airflow submission. It handles small JSON/YAML
  payloads and metadata.
- **Data plane** — isolated runner processes, datasource adapters, data engines,
  and artifact storage. It handles credentials, compute, and large datasets.

Airflow remains the durable orchestrator. It receives a workflow/run reference,
not dynamically generated Python source for every YAML file. A stable Airflow DAG
or operator invokes the independent runner, which loads the versioned definition
and execution plan.

Open the [interactive architecture diagram](architecture/transformation-dsl-platform.html)
and switch between its request, execution, and extensibility views.

## Request and execution lifecycle

1. A client submits a YAML definition to the REST API.
2. The API enforces authentication, size limits, and request tracing.
3. The parser converts safe YAML into a versioned `WorkflowSpec`.
4. The compiler resolves references, validates dependencies and conditions, detects
   cycles, and emits an immutable `ExecutionPlan` with a content fingerprint.
5. The workflow catalog stores definitions and plans; run metadata stores state.
6. The Airflow backend submits only stable identifiers and idempotency metadata.
7. A runner loads the plan and selects datasource and engine plug-ins.
8. Large data stays in Snowflake, Databricks, Oracle, Blob, object storage, or a
   worker-local engine. Steps exchange `DatasetRef` metadata instead of moving
   frames through the API or Airflow XCom.
9. The runner records step/run state and writes durable output artifacts.

## Dependency rule

Dependencies point inward:

```text
domain                 no framework or vendor dependency
application            domain ports and use cases
dsl                    domain model implementation
plugins                domain port discovery and registration
adapters                vendor/framework implementations of domain ports
api / cli              inbound delivery mechanisms
bootstrap              application composition root
```

The domain never imports FastAPI, Airflow, Spark, database drivers, or cloud SDKs.
Adapters may import the domain, but the domain does not import adapters.

## Repository layout

```text
.
├── docs/
│   ├── ARCHITECTURE.md
│   ├── PLUGINS.md
│   └── architecture/
├── src/transformation_dsl/
│   ├── adapters/
│   │   ├── datasources/
│   │   ├── engines/
│   │   ├── orchestrators/
│   │   └── persistence/
│   ├── api/
│   ├── application/
│   ├── domain/
│   ├── dsl/
│   ├── plugins/
│   ├── bootstrap.py
│   ├── cli.py
│   └── config.py
├── tests/
├── workflows/
├── Dockerfile
├── compose.yaml
└── pyproject.toml
```

## Core contracts

- `DslParser` — turns raw YAML into a `WorkflowSpec`.
- `DslCompiler` — turns the model into a deterministic `ExecutionPlan`.
- `WorkflowRepository` — stores immutable workflow versions and compiled plans.
- `RunRepository` — provides idempotent run state and optimistic transitions.
- `ExecutionBackend` — submits/cancels work; Airflow is one implementation.
- `PlanExecutor` — executes a plan inside the data-plane runner.
- `DatasourceAdapter` — reads/writes through portable `DatasetRef` values.
- `EngineAdapter` — invokes a named transformation using an engine-specific
  context.
- `ArtifactStore` — stores plans, logs, manifests, and durable output metadata.

## DSL compilation boundaries

YAML is data, not executable Python. The compiler should:

- load YAML with a safe loader and reject aliases or constructs outside policy;
- validate against a versioned Pydantic model before resolving anything;
- parse `${...}` references into a small typed reference grammar;
- parse `when` and pre-validation expressions into an allow-listed expression AST;
- never call Python `eval`;
- verify that each dependency and step output exists;
- detect dependency cycles;
- validate that `uses` is allow-listed or registered;
- normalize output names and produce a stable plan fingerprint;
- preserve the original source location for useful errors.

Conditions should be evaluated at step runtime after dependencies have completed.
An unmet `when` marks a step `skipped`; a failed pre-validation marks it `failed`
with a structured error. Dataset selection belongs in an explicit expression or
step argument, not hidden orchestration code.

## Scalability and robustness

- Keep the API stateless and horizontally scalable.
- Use PostgreSQL for workflow and run metadata.
- Use object storage for immutable plans, manifests, logs, and large artifacts.
- Use one Airflow submission per workflow run; avoid one dynamically registered
  Airflow DAG per uploaded YAML file.
- Package heavy engines and vendor SDKs in dedicated worker images.
- Apply concurrency, memory, time, and dataset-size limits per tenant or queue.
- Make submit requests idempotent and step retries safe.
- Use a transactional outbox when metadata persistence and Airflow submission
  must be coordinated.
- Propagate request ID, run ID, plan fingerprint, workflow version, and step ID
  through logs, traces, and metrics.
- Keep credentials in a secret manager or Airflow connection, referenced by a
  logical connection name; never embed secrets in YAML.
- Authorize workflow functions, connections, and engines independently.

## Recommended delivery milestones

1. Safe YAML parsing, schema validation, graph validation, and deterministic plans.
2. Local in-process runner with a small built-in function allow-list.
3. Run/workflow repositories and object artifact storage.
4. One thin Airflow execution-backend plug-in and a stable runner DAG/operator.
5. One lightweight engine plus one datasource adapter end to end.
6. Authentication/authorization, idempotency, quotas, audit events, and telemetry.
7. Add Spark and vendor adapters based on real workload needs.

Each milestone should add contract tests that every new adapter must pass.
