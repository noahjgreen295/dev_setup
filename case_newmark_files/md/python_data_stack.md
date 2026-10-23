# The Data Engineering Stack

Data only flows one way. It starts as rows in someone else's system, gets captured and landed, settles into object storage as files, gets organized into tables, crunched by an engine, modeled into something trustworthy, and finally served to the people and machines that make decisions with it. Orchestration, quality checks and lineage run alongside the whole way.

Part of [the Sprawl](README.md). State as of October 2026. For the styled, interactive version, open [python_data_stack.html](../html/python_data_stack.html) in a browser.

**Status key:** **PICK** = modern default · alt = solid alternative · ⚠️ edge = maintained, but teetering, too new, or overtaken · ◆×N = appears in N places

## Default builds

- **Small-team lakehouse:** dlt → S3 → Parquet → DuckLake or Iceberg → DuckDB / MotherDuck → Polars → dbt → Dagster → Superset
- **Enterprise cloud warehouse:** Fivetran → Snowflake or Databricks → dbt → Airflow 3 → semantic layer → BI tool
- **Real-time pipeline:** Debezium → Kafka → Flink → Iceberg → Trino / ClickHouse

---

## L9 · Sources & capture

Where data is born: application databases, SaaS tools, event streams, files dropped by partners. Capture means getting changes out without hurting the source system.

**Pick:** Change data capture (CDC) with Debezium for databases; vendor APIs for SaaS

| Tool | Status | Notes |
|---|---|---|
| Debezium | **PICK** | Reads a database's transaction log and emits every insert, update and delete as an event. The open-source CDC standard. |
| PostgreSQL / MySQL | **PICK** | The most common source systems. Read from a replica or the logical replication stream, never by hammering production with full-table scans. |
| SaaS APIs | **PICK** | Salesforce, Stripe, HubSpot and friends. Rate limits and schema drift are the main enemies. |
| Event tracking | alt | Snowplow, Segment, RudderStack. Behavioral events from apps and websites. |
| Files & SFTP drops | alt | Partner CSVs and Excel files. Validate hard on arrival; this is where garbage enters. |

---

## L8 · Ingestion (EL)

Extract and load: move data from sources into your storage, ideally untouched. Buy it for common SaaS sources, write it in Python for everything else.

**Pick:** dlt for code-first Python; Fivetran when you'd rather pay than maintain

| Tool | Status | Notes |
|---|---|---|
| dlt | **PICK** | Python library: write a generator, get schema inference, incremental loading and normalized tables in any destination. Runs anywhere Python runs. |
| Fivetran | **PICK** | Managed connectors for hundreds of sources. Expensive at volume, but zero maintenance. Now also owns dbt. |
| Airbyte | alt | Open-source connector platform you can self-host. Big catalog; heavier to operate than dlt. |
| Kafka Connect | alt | Connectors that stream data into and out of Kafka. Debezium runs on it. |
| Meltano / Singer | ⚠️ edge | The open tap-and-target standard. Still maintained, but connector quality varies and newer tools have taken its momentum. |

---

## L7 · Streaming

For data that can't wait for the nightly batch: a durable log of events, and processors that compute on them as they arrive. Most teams need less of this than they think.

**Pick:** Kafka as the log; Flink for heavy stream processing

| Tool | Status | Notes |
|---|---|---|
| Apache Kafka | **PICK** | The durable, replayable event log everyone integrates with. Managed via Confluent, MSK and others. |
| Redpanda | alt | Kafka-compatible API, single binary, no JVM. Simpler to run yourself. |
| Apache Flink / PyFlink | **PICK** | The heavyweight stream processor: event time, exactly-once, huge state. PyFlink exposes it to Python. |
| Spark Structured Streaming ◆×3 | alt | Streaming with the same DataFrame API as batch Spark. Natural if you're already on Databricks. |
| Quix Streams | alt | Pure-Python stream processing on Kafka. Lightweight, no cluster to run. |
| Bytewax | ⚠️ edge | Python dataflow streaming with a Rust core. No release since November 2024. |

---

## L6 · Object storage

The cheap, practically infinite disk at the bottom of every modern data platform. Everything above stores its files here.

**Pick:** Your cloud's object store: S3, GCS or Azure Blob/ADLS

| Tool | Status | Notes |
|---|---|---|
| S3 / GCS / ADLS | **PICK** | Durable, cheap, and the de facto API everyone speaks (S3's in particular). |
| Cloudflare R2 | alt | S3-compatible with no egress fees. Attractive when data is read from many places. |
| MinIO / SeaweedFS | alt | Self-hosted S3-compatible storage for on-prem or local development. Check MinIO's AGPL license and edition terms before adopting. |
| HDFS | ⚠️ edge | Hadoop's distributed filesystem. Still running in many enterprises; nobody starts new platforms on it. |

---

## L5 · File formats

How bytes are laid out on disk and in memory. Columnar formats let engines read only the columns a query needs, which is most of the speed.

**Pick:** Parquet on disk, Arrow in memory

| Tool | Status | Notes |
|---|---|---|
| Parquet | **PICK** | Columnar, compressed, typed. The universal analytical file format. |
| Apache Arrow | **PICK** | The shared in-memory columnar format. Lets Polars, DuckDB, pandas and Spark pass data with zero copying. |
| Avro | alt | Row-based with embedded schemas. The usual choice for Kafka messages and schema registries. |
| CSV / JSON | alt | For landing raw data only. Convert to Parquet as soon as it's in. |
| ORC | ⚠️ edge | Hive's columnar format. Still readable everywhere, but Parquet won. |

---

## L4 · Table formats & catalogs

Turns a pile of Parquet files into real tables: transactions, schema evolution, time travel, safe concurrent writes. The catalog tracks where each table lives.

**Pick:** Apache Iceberg with a REST catalog

### Table format

| Tool | Status | Notes |
|---|---|---|
| Apache Iceberg | **PICK** | The open standard most engines now read and write: Snowflake, BigQuery, Databricks, Trino, Spark, DuckDB. Spec v3 adds deletion vectors and new types. |
| Delta Lake | alt | Databricks' format. The natural pick inside Databricks; interop with Iceberg keeps improving. |
| DuckLake | alt | DuckDB's lakehouse format, 1.0 since April 2026. Keeps metadata in a SQL database (Postgres, SQLite or DuckDB) instead of files. Simple and fast for small teams. |
| Apache Hudi | ⚠️ edge | Strong at upserts and streaming ingestion, but losing mindshare to Iceberg and Delta. |

### Catalog

| Tool | Status | Notes |
|---|---|---|
| Iceberg REST catalog | **PICK** | The standard catalog API: Apache Polaris, Lakekeeper, AWS Glue, Snowflake Open Catalog all speak it. |
| Unity Catalog | alt | Databricks' catalog, open-sourced. Governance plus Delta and Iceberg. |
| Hive Metastore | ⚠️ edge | The original Hadoop-era catalog. Many tools still support it; new platforms use REST catalogs. |

---

## L3 · Query engines & warehouses

The SQL engines that run the big queries over stored data. Managed warehouses trade money for zero operations; open engines trade operations for control.

**Pick:** A managed warehouse at scale; DuckDB / MotherDuck when data fits on one big machine

### Managed

| Tool | Status | Notes |
|---|---|---|
| Snowflake | **PICK** | Separate storage and compute, near-zero tuning, strong SQL. Python via Snowpark. |
| BigQuery | **PICK** | Serverless, pay per query. The default on Google Cloud. |
| Databricks | **PICK** | Spark-based lakehouse platform; strongest when ML and data engineering share one place. |
| Redshift | alt | AWS's warehouse. Fine inside AWS; rarely chosen fresh over the three above. |

### Open

| Tool | Status | Notes |
|---|---|---|
| DuckDB / MotherDuck ◆×2 | **PICK** | In-process analytics engine: queries Parquet and Iceberg directly, absurdly fast on one machine. MotherDuck is the hosted version. |
| Trino | alt | Distributed SQL across many sources at once. The open engine for big lakehouses. |
| ClickHouse | alt | Blazing real-time analytics on event data. Powers many user-facing dashboards. |
| Apache Spark ◆×3 | alt | The distributed workhorse for batch at true big-data scale. Spark 4 is current. |

---

## L2 · Python compute

Dataframes and distributed compute from Python, for the transformations that SQL handles badly: complex logic, ML features, API calls in the loop.

**Pick:** Polars on one machine; PySpark on a cluster

| Tool | Status | Notes |
|---|---|---|
| Polars | **PICK** | Rust dataframe library: lazy queries, multi-core by default, often 10x pandas. The modern single-node default. |
| DuckDB (Python) ◆×2 | **PICK** | SQL over dataframes, Parquet and Arrow with zero setup. Pairs perfectly with Polars. |
| pandas | alt | Still everywhere and every library accepts it. Fine for small data; slow and memory-hungry beyond that. |
| PySpark ◆×3 | **PICK** | Python on Spark for cluster-scale batch. Includes the pandas API on Spark. |
| Ibis | alt | One dataframe API that compiles to DuckDB, Polars, Snowflake, BigQuery and more. Write once, move engines later. |
| Ray / Dask | alt | Distributed Python beyond dataframes. Ray dominates ML-heavy work; Dask scales pandas-style code. |
| Daft | alt | Distributed dataframes built for multimodal data: images, documents, embeddings. |

---

## L1 · Transformation & modeling

Turning raw tables into clean, tested, documented models (staging, intermediate, marts) with dependencies tracked automatically. This is where analytics engineering lives.

**Pick:** dbt Core

| Tool | Status | Notes |
|---|---|---|
| dbt Core | **PICK** | SQL models with refs, tests and docs. The industry standard. Core 2.0 on the Rust Fusion engine is in alpha; 1.x remains the stable line. |
| SQLMesh | alt | dbt's sharpest rival: understands your SQL, plans changes before applying them, cheap virtual environments. Also owned by Fivetran now. |
| SQLGlot | alt | Pure-Python SQL parser and transpiler across 20+ dialects. Powers SQLMesh and many other tools. |
| Dataform | alt | Google's dbt-like tool, built into BigQuery. |
| Python models | alt | dbt Python models or plain Polars jobs for the steps SQL can't express. Keep them the exception. |

---

## L0 · Serving & consumption

Where data finally earns its keep: dashboards, metrics everyone agrees on, data pushed back into operational tools, features for models, and APIs.

**Pick:** A semantic layer feeding your BI tool

| Tool | Status | Notes |
|---|---|---|
| Semantic layer | **PICK** | dbt Semantic Layer (MetricFlow) or Cube. Define "revenue" once so every dashboard computes it the same way. |
| Superset / Metabase | **PICK** | Open-source BI. Metabase is easier for business users; Superset is more powerful. |
| Looker / Power BI / Tableau | alt | The commercial BI incumbents. Usually chosen by the org, not the engineer. |
| Reverse ETL | alt | Census, Hightouch: push modeled data back into Salesforce, ad platforms and support tools. |
| Feast | alt | Feature store serving model features consistently for training and live inference. |
| Streamlit / marimo | alt | Quick internal data apps and reactive notebooks in pure Python. |
| Data APIs | alt | FastAPI over DuckDB or the warehouse, for programmatic consumers. See the Python API Stack. |

---

## Sidecars

The control plane. These don't move data themselves; they schedule it, check it, and record where it came from. A pipeline without them works until the first silent failure, then nobody trusts the numbers.

### Orchestration

| Tool | Status | Notes |
|---|---|---|
| Apache Airflow 3 | **PICK** | Still the most deployed orchestrator. Version 3 added data-aware assets, DAG versioning and a new UI; 3.3 is current. Every cloud offers a managed version. |
| Dagster | **PICK** | Asset-based orchestration: you declare tables, it works out the runs. Excellent local development, lineage and dbt integration. The modern favorite for new platforms. |
| Prefect | alt | Turn any Python function into a tracked, retryable flow. Lightest path from script to pipeline. |
| Kestra | alt | Declarative YAML pipelines, language-agnostic. Popular with mixed teams. |
| Luigi | ⚠️ edge | Spotify's early orchestrator. Still maintained, rarely chosen for new work. |

### Quality & contracts

| Tool | Status | Notes |
|---|---|---|
| Pandera | **PICK** | Schema validation for Polars, pandas and PySpark dataframes, written like Pydantic models. |
| dbt tests | **PICK** | not_null, unique, relationships and custom SQL tests, right next to the models. |
| Great Expectations (GX Core) | alt | Expectation suites with rich data docs. Powerful; heavier setup than Pandera. |
| Soda | alt | Data quality checks in a YAML language, with monitoring and alerts. |
| Pydantic | alt | Validate records at ingestion boundaries, especially API and JSON sources. |

### Lineage & metadata

| Tool | Status | Notes |
|---|---|---|
| OpenLineage | **PICK** | Open standard for lineage events, emitted by Airflow, Spark, dbt and others. |
| DataHub / OpenMetadata | alt | Searchable data catalogs: who owns this table, where did it come from, who reads it. |
| Marquez | alt | Reference OpenLineage server and UI. Good starting point. |

### Ship & run

| Tool | Status | Notes |
|---|---|---|
| Terraform / OpenTofu | **PICK** | Infrastructure as code for buckets, warehouses, permissions and networks. |
| Docker + uv | **PICK** | Same build story as any Python service: locked dependencies, slim images. |
| Kubernetes / managed runners | alt | Run orchestrator workers and jobs. Most teams are better served by a managed service. |
| CI for data | **PICK** | Run dbt builds and data tests on every pull request against a sample or a zero-copy clone. |

---

## Rules of the road

- **ELT, not ETL.** Land raw data first, untouched, then transform inside the warehouse or lake. You can always rebuild from raw; you can't un-transform.
- **Idempotent, partitioned loads.** Running a job twice must give the same result as running it once. Overwrite a date partition rather than appending blindly.
- **Parquet, not CSV.** Typed, compressed and columnar. CSV belongs only at the landing edge, when a source gives you nothing better.
- **Contracts at the boundaries.** Validate schemas where data enters and where it leaves. Inside, trust your models.
- **Think in assets, not tasks.** Orchestrate the tables you produce and their dependencies, not just a list of scripts to run in order.

## The Fivetran factor

Fivetran acquired Tobiko Data (SQLMesh and SQLGlot) in September 2025, then completed its merger with dbt Labs on June 1, 2026. The two leading transformation frameworks and the most widely used Python SQL parser now share one owner. dbt Core 2.0, built on the Rust Fusion engine, is in alpha under Apache 2.0, and everything stays open source for now. The practical hedge: keep your models in plain SQL with minimal framework-specific magic, so moving between dbt and SQLMesh stays cheap.

## Dog Solitude

The scrapyard. Dead, retired, renamed or abandoned tools, by layer. Mostly the Hadoop era.

| Code | Short | Layer | Obsolete |
|---|---|---|---|
| L8 | ingest | Ingestion (EL) | Apache Sqoop (retired to the Apache Attic) |
| L7 | stream | Streaming | Faust (original Robinhood release), Spark Streaming DStreams |
| L3 | engines | Query engines & warehouses | Hadoop MapReduce jobs, Apache Pig, Hive on MapReduce |
| L2 | compute | Python compute | Koalas (merged into PySpark), pandas-profiling (renamed ydata-profiling) |
| — | sidecar | Orchestration | Apache Oozie, Azkaban, Airflow 1.x |
| — | sidecar | Quality & contracts | Great Expectations 0.x API |

## Sources

- [Fivetran and dbt Labs complete merger (Fivetran)](https://www.fivetran.com/press/fivetran-dbt-labs-complete-merger-to-create-the-data-infrastructure-for-trusted-ai-agents)
- [Fivetran acquires Tobiko Data (Business Wire)](https://www.businesswire.com/news/home/20250903275106/en/Fivetran-Acquires-Tobiko-Data-to-Power-the-Next-Generation-of-Advanced-AI-Ready-Data-Transformation)
- [Fivetran–dbt merger and Fusion engine (Kestra)](https://kestra.io/resources/data/fivetran-dbt-merger-fusion-engine)
- [Airflow 3 release history (versionlog)](https://versionlog.com/apache-airflow/3/)
- [DuckLake 1.0 announcement](https://ducklake.select/2026/04/13/ducklake-10/)
- [bytewax on PyPI](https://pypi.org/project/bytewax)
