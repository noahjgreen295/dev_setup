# The Sprawl

Field guides to the Python ecosystem. Each guide charts one part of the Python world, names the modern default at every layer, and keeps a Dog Solitude for the tools that didn't make it. Mapped October 2026.

For the styled hub, open [sprawl_map.html](sprawl_map.html) in a browser.

## Charted sectors

| Code | Guide | Shape | Tools | Dead | Signature picks |
|---|---|---|---|---|---|
| TCS | [Python Toolchain Stack](python_toolchain_stack.md) | 10 layers | 68 | 41 | uv, Ruff, Pyright |
| DOC | [Python Docs Switchboard](python_docs_switchboard.md) | 4 routes | 48 | 9 | Sphinx + MyST, OpenAPI 3.1, Jupyter Book 2 |
| API | [Python API Stack](python_api_stack.md) | 10 layers + 4 sidecars | 88 | 16 | FastAPI, Pydantic, PostgreSQL |
| DAT | [Data Engineering Stack](python_data_stack.md) | 10 layers + 4 sidecars | 76 | 12 | dlt, Iceberg, dbt |
| DEP | [Python Deployment Stack](python_deploy_stack.md) | 10 layers + 3 sidecars | 63 | 16 | python:slim, Cloud Run, GitHub Actions |

Each guide also has a styled HTML version with the same name.

## Uncharted

- **DFX · Dataframe Switchboard:** pandas, Polars, DuckDB, Spark, routed by data size.
- **QNT · Quant Stack:** numerics, performance, market data, pricing and backtesting.
- **CON · Concurrency Switchboard:** threads, asyncio, processes and free-threading, routed by workload.
- **MLO · ML / MLOps Stack:** experiments, registries, serving and monitoring.

## Shared hardware

Tools that cross sector lines. The more sectors a tool appears in, the more worth knowing deeply it is.

| Tool | Role | TCS | DOC | API | DAT | DEP |
|---|---|:-:|:-:|:-:|:-:|:-:|
| uv | Installs, locks, builds, runs | ● | | ● | ● | ● |
| Pydantic | Validation everywhere data crosses a boundary | ● | ● | ● | ● | ● |
| Docker | Shipping it | | | ● | ● | ● |
| pytest | Tests, including doctests | ● | ● | ● | | |
| FastAPI | Services and data APIs | | ● | ● | ● | |
| Kafka | Event streams and async contracts | | ● | ● | ● | |
| PostgreSQL | Source system and datastore | | | ● | ● | ● |
| Ruff | Lint, format, docstring checks | ● | ● | | | |
| OpenAPI | The contract for HTTP services | | ● | ● | | |
| DuckDB | Analytics in-process | | | ● | ● | |
| Terraform | Infrastructure as code | | | | ● | ● |
| OpenTelemetry | Traces, metrics and logs | | | ● | | ● |
| Caddy | Edge proxy with automatic HTTPS | | | ● | | ● |
| ~~Swagger 2.0~~ | Dead: superseded by OpenAPI 3 | | ● | ● | | |
| ~~pydocstyle~~ | Dead: folded into Ruff | ● | ● | | | |

## Also in this folder

- [python_obsolete_practices.md](python_obsolete_practices.md): condensed notes on outdated Python coding patterns and their replacements.
