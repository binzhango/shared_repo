# Architecture

## Goal

This repository is an independent, reusable transformation library and service.
Other services submit versioned YAML workflows through FastAPI. The platform
validates and compiles each workflow into a framework-neutral execution plan,
delegates durable scheduling to Airflow, and lowers the plan through Hamilton,
the native runtime, or another `DagRuntimeAdapter`.

This milestone is a scaffold: the package boundaries, domain contracts, extension
points, API shape, container, and tests exist; parsing, compilation, persistence,
Airflow submission, Hamilton lowering, and vendor operations intentionally do not.

## Architecture choice

Start as a modular monolith with separately deployable control-plane and runner
processes. This preserves strong module boundaries without creating an operational
network hop for every class. Split a module into a service only when scale,
security, ownership, or release cadence requires it.

The two runtime planes are:

- **Control plane** — FastAPI, workflow catalog, DSL parser/compiler, adapter
  catalog, run metadata, and Airflow submission. It handles small JSON/YAML
  payloads and metadata.
- **Data plane** — isolated runner processes, a selected DAG runtime, reusable
  datasource/engine adapters, and artifact storage. It handles credentials,
  compute, and large datasets.

Airflow remains the durable orchestrator. It receives a workflow/run reference,
not dynamically generated Python source for every YAML file. A stable Airflow DAG
or operator invokes the independent runner, which loads the versioned definition
and canonical execution plan. The runner selects an execution profile and asks its
`DagRuntimeAdapter` to lower and execute the plan.

Open the [interactive architecture diagram](architecture/transformation-dsl-platform.html)
and switch between its request, execution, and extensibility views.

## Request and execution lifecycle

1. A client submits a YAML definition to the REST API.
2. The API enforces authentication, size limits, and request tracing.
3. The parser converts safe YAML into a versioned `WorkflowSpec`.
4. The compiler resolves references, validates dependencies and conditions,
   detects cycles, and emits an immutable, framework-neutral `ExecutionPlan`
   with a content fingerprint.
5. The workflow catalog stores definitions and plans; run metadata stores state.
6. The Airflow backend submits only stable identifiers and idempotency metadata.
7. A runner resolves the workflow's named execution profile.
8. The selected DAG runtime—Hamilton, native, or future framework—lowers the
   canonical plan without re-parsing YAML or redefining DSL semantics.
9. Runtime nodes invoke the shared datasource and engine adapter modules.
10. Large data stays in Snowflake, Databricks, Oracle, Blob, object storage, or a
   worker-local engine. Steps exchange `DatasetRef` metadata instead of moving
   frames through the API or Airflow XCom.
11. The runner records step/run state and writes durable output artifacts.

## Dependency rule

Dependencies point inward:

```text
domain                 no framework or vendor dependency
application            domain ports and use cases
dsl                    domain model implementation
plugins                domain port discovery and registration
adapters                reusable vendor/framework implementations of domain ports
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
│   ├── ADAPTERS.md
│   └── architecture/
├── src/transformation_dsl/
│   ├── adapters/
│   │   ├── datasources/
│   │   ├── engines/
│   │   ├── orchestrators/
│   │   ├── runtimes/
│   │   └── persistence/
│   ├── api/
│   ├── application/
│   ├── domain/
│   ├── dsl/
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
- `OrchestrationBackend` — submits/cancels complete runs; Airflow is one
  implementation.
- `DagRuntimeAdapter` — lowers and executes the canonical plan; Hamilton and the
  native runtime are initial implementations.
- `DatasourceAdapter` — reads/writes through portable `DatasetRef` values.
- `EngineAdapter` — invokes a named transformation using an engine-specific
  context.
- `ArtifactStore` — stores plans, logs, manifests, and durable output metadata.

## Framework portability boundary

The compiler owns DSL meaning and produces one canonical intermediate
representation:

```text
YAML
  → WorkflowSpec
  → canonical ExecutionPlan
  → DagRuntimeAdapter
      ├── HamiltonDagRuntimeAdapter
      ├── NativeDagRuntimeAdapter
      └── future framework adapter
```

A runtime adapter must not parse YAML, resolve `${...}` references, reinterpret
`when`, or invent retry behavior. It only lowers already validated plan nodes into
its framework representation and executes them. Capability checks reject plans
that cannot be represented faithfully.

Workflow definitions refer to a stable `execution_profile`. Deployment
configuration maps that name to an orchestrator, DAG runtime, and namespaced
runtime options. Switching frameworks therefore changes configuration rather
than hundreds of workflow files.

Hamilton is a DAG runtime, not a data engine. It coordinates fine-grained
function dependencies inside the runner; Spark, Pandas, Polars, and DuckDB still
perform data work through `EngineAdapter`.

## Reusable common adapters

Snowflake, Databricks, Oracle, Blob, Spark, Pandas, Polars, DuckDB, Hamilton, and
Airflow adapters live as modules in this distribution. Other projects import and
reuse the registry or individual adapters instead of producing a package per
integration.

Heavy vendor dependencies remain optional extras and must be imported lazily.
This permits one maintained codebase while allowing control-plane and worker
images to install different dependency sets. Python entry points remain available
only for genuinely private or uncommon extensions.

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
- Reuse the same adapter modules while installing heavy engines and vendor SDK
  extras only in the worker images that need them.
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
2. Native runtime plus the shared runtime conformance test suite.
3. Run/workflow repositories and object artifact storage.
4. Hamilton lowering adapter and a thin Airflow orchestration backend.
5. One lightweight engine plus one shared datasource adapter end to end.
6. Authentication/authorization, idempotency, quotas, audit events, and telemetry.
7. Add Spark and vendor adapters based on real workload needs.

Each milestone should add contract tests that every new adapter must pass.
