# Plug-in model

Datasource adapters, data engines, and execution backends are installed packages,
not branches in the compiler. The scaffold discovers them through standard Python
package entry points.

## Entry-point groups

| Group | Contract | Example names |
|---|---|---|
| `transformation_dsl.datasources` | `DatasourceAdapter` | `snowflake`, `oracle`, `azure_blob` |
| `transformation_dsl.engines` | `EngineAdapter` | `pandas`, `polars`, `duckdb`, `spark` |
| `transformation_dsl.execution_backends` | `ExecutionBackend` | `airflow`, `in_process` |

An integration should live in a separate package when it brings a large SDK,
native dependency, separate release cadence, or credentials that should not be
present in every worker image.

## Example package metadata

```toml
[project]
name = "acme-dsl-duckdb"
dependencies = [
  "transformation-dsl-platform>=0.1,<0.2",
  "duckdb>=1,<2",
]

[project.entry-points."transformation_dsl.engines"]
duckdb = "acme_dsl_duckdb:DuckDbEngine"
```

The entry point may expose an adapter instance or a zero-argument adapter class.
Names must be unique within a process.

## Adapter expectations

Every adapter should:

- implement only its domain port;
- expose a stable `name`;
- validate configuration without opening a data connection;
- accept logical secret/connection references instead of raw credentials;
- emit structured errors that distinguish retryable from permanent failures;
- propagate run and step correlation fields;
- support cancellation and cleanup where its underlying system permits;
- declare thread/process safety and supported data formats;
- pass a shared contract-test suite.

The registry currently performs discovery and duplicate-name protection only.
Configuration validation, health checks, compatibility metadata, and contract
test fixtures belong in a later milestone.

## Avoiding the connector × engine matrix

Datasource adapters return `DatasetRef` metadata. Engines decide how to consume
that reference efficiently. For example, Spark can read a table URI directly,
while a local Polars worker can materialize a bounded Arrow or Parquet artifact.
This prevents every datasource package from implementing a custom path for every
engine.
