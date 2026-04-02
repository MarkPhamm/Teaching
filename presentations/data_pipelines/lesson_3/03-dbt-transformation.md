# dbt & the Transformation Layer

## 1. Why dbt Exists

### 1.1 The Problem

Before dbt, analytics teams wrote SQL transformations as scattered scripts — no version control, no testing, no documentation. When something broke, nobody knew which script to fix or what downstream dashboards were affected.

The core issue: SQL transformations were treated as disposable queries, not as software that needed the same rigor as application code.

### 1.2 What dbt Does

dbt (data build tool) applies software engineering practices to SQL transformations. You write SELECT statements, and dbt handles everything else — creating tables/views, resolving dependencies, running tests, and generating documentation.

It operates entirely inside the warehouse. It doesn't extract or load data. It reads from tables that already exist and writes the results back to the same warehouse. All compute happens on the warehouse's engine.

### 1.3 dbt Core vs dbt Cloud

dbt Core is the open-source CLI. It's free, runs from your terminal or CI/CD pipeline, and pairs with any orchestrator. dbt Cloud is a paid SaaS layer on top — web IDE, scheduling, alerting. Same project structure, same SQL.

We use dbt Core because it gives full control over the execution environment and integrates cleanly with orchestrators like Airflow or Dagster.

---

## 2. Project Structure

A dbt project has a predictable anatomy. The key directories:

- **models/** — the heart of the project. Each .sql file is a model (a SELECT statement).
- **tests/** — custom data quality assertions.
- **macros/** — reusable Jinja/SQL functions.
- **seeds/** — small CSV files loaded as tables (lookup tables, mappings).
- **snapshots/** — SCD Type 2 tracking for slowly changing dimensions.

YAML files live alongside models. They define sources (raw tables), tests, descriptions, and column metadata. Think of them as the contract for each model — what it is, what it promises, and where its data comes from.

Two critical config files:
- **dbt_project.yml** — project-level settings: name, default materializations per folder, variable defaults.
- **profiles.yml** — warehouse connection credentials. Lives outside the project repo (~/.dbt/) because it contains secrets.

---

## 3. Models & Layers

### 3.1 What is a Model?

A model is a .sql file with a single SELECT statement. dbt compiles it (resolving Jinja and references) and materializes it as a table or view in the warehouse. You never write DDL — no CREATE TABLE, no INSERT INTO.

### 3.2 The Three Layers

The standard dbt project organizes models into three layers, each with a specific purpose:

**Staging (stg_)** — one model per source table. Light transformations only: renaming columns, casting types, reformatting dates. No joins. The goal is a clean, consistent interface over raw data. Materialized as views because they're lightweight and always reflect the latest source data.

**Intermediate (int_)** — where business logic lives. Joins staging models together, applies filters, aggregations, pivots. These models exist to break complex logic into manageable steps. Not exposed to end users.

**Marts (fct_, dim_)** — the final outputs that BI tools and analysts consume. Facts (fct_) represent events and transactions. Dimensions (dim_) represent entities and attributes. Materialized as tables for fast reads.

Why this separation matters: when a source schema changes, you update one staging model. The intermediate and mart layers don't need to know about raw table structure — they only reference staging models.

### 3.3 Naming Conventions

Staging models follow the pattern `stg_<source>__<table>` (double underscore). This tells you exactly where the data came from. Mart models use `fct_` or `dim_` prefixes to signal their role.

Name models after what they represent, not what they do. "transform_orders" tells you nothing — "stg_shopify__orders" tells you the source and the entity.

---

## 4. Materializations

Materializations determine how dbt writes results to the warehouse. The choice affects query performance, storage cost, and build time.

**View** — a saved query that re-executes every time it's read. No storage cost, always fresh, but slow if the underlying query is heavy. Best for staging models.

**Table** — physically stored data, rebuilt from scratch on every run. Fast reads, but the full rebuild can be slow for large datasets. Best for mart models.

**Incremental** — only processes new or changed rows. On the first run, it builds the full table. On subsequent runs, it appends or upserts only what's new. Best for large, append-heavy tables (event logs, clickstreams). Requires a strategy for identifying new rows (usually a timestamp) and a unique_key for upsert behavior.

**Ephemeral** — not stored anywhere. Injected as a CTE into downstream models. Useful for DRY reusable logic that doesn't need to be queried directly.

The default strategy: staging = view, marts = table. Only upgrade to incremental when table rebuilds become too slow. Don't use incremental by default — it adds complexity (handling late-arriving data, schema changes, full refreshes).

---

## 5. Sources & ref()

### 5.1 Sources

Sources declare the raw tables that exist in the warehouse before dbt touches them. You define them in YAML, and reference them with `source('name', 'table')`.

Why bother? Four reasons:
1. **Lineage** — dbt tracks which raw tables feed into which models.
2. **Freshness** — you can set thresholds and alert when source data is stale.
3. **Documentation** — sources appear in the DAG and docs site.
4. **Abstraction** — if a schema name changes, update one YAML file instead of every model that references it.

### 5.2 ref()

ref() is the most important function in dbt. It references another dbt model by name, and dbt resolves it to the actual schema.table in the warehouse.

What ref() does beyond name resolution: it builds the DAG. Every ref() call creates a dependency edge. dbt uses this to determine execution order — staging models run before intermediate, intermediate before marts.

ref() is also environment-aware. In dev, it resolves to your dev schema. In prod, it resolves to prod. No code changes needed.

Never hardcode table names. Always use ref() for dbt models and source() for raw tables. Hardcoded names break lineage tracking and environment switching.

### 5.3 The DAG

The DAG (Directed Acyclic Graph) is the dependency graph dbt builds from all your ref() and source() calls. It determines what to build and in what order. It also powers lineage visualization — you can see, for any model, what feeds into it and what depends on it.

---

## 6. Testing & Documentation

### 6.1 Built-in Tests

dbt has four generic tests you apply in YAML:
- **not_null** — no NULL values in the column.
- **unique** — every value is distinct (validates primary keys).
- **accepted_values** — column only contains values from your allowlist.
- **relationships** — foreign key check (every value exists in the referenced model).

At minimum, every model should have unique + not_null on its primary key. This catches duplicates and missing data before they propagate downstream.

### 6.2 Custom Tests

Any SQL query can be a test. Put a .sql file in the tests/ directory. If the query returns rows, the test fails. This handles business rules that go beyond the four generic tests — revenue should never be negative, order totals should reconcile, etc.

### 6.3 Documentation

Documentation lives in YAML alongside the models. Add descriptions to models and columns. Run `dbt docs generate` to build a browsable site with model descriptions, column details, test coverage, and an interactive lineage graph.

The key practice: update docs in the same PR as the model change. Documentation that lives separately from code drifts out of date immediately.

---

## 7. Jinja, Macros & Packages

### 7.1 Jinja

dbt uses Jinja templating inside SQL files. Expressions (`{{ }}`) output values. Statements (`{% %}`) add control flow — if/else, for loops, variables.

Common uses: referencing models, environment-specific logic (LIMIT in dev), generating repetitive SQL.

The trap: Jinja makes SQL harder to read. Use it for genuinely repetitive patterns, not to build a programming language inside SQL. If someone can't understand the model by reading it, the Jinja is too complex.

### 7.2 Macros

Macros are reusable Jinja functions defined in the macros/ directory. They generate SQL snippets — unit conversions, standard cleaning patterns, dynamic pivots.

Use macros when the same logic appears in multiple models. Don't create a macro for logic used only once — that's abstraction for abstraction's sake.

### 7.3 Packages

Packages are shared dbt projects with pre-built macros, tests, and models. Install them via packages.yml and `dbt deps`.

**dbt_utils** is essential — surrogate key generation, pivot, union_relations, date_spine. **codegen** auto-generates staging models and YAML schemas from your sources, saving hours of boilerplate.

---

## 8. Best Practices

- **One model, one purpose** — don't cram multiple transforms into one file.
- **Always use ref() and source()** — never hardcode table names.
- **Test primary keys** — unique + not_null on every model.
- **Follow the layer pattern** — staging → intermediate → marts.
- **Document as you build** — descriptions in the same PR as the model.
- **Don't skip staging** — marts that read directly from raw tables are fragile.
- **Don't over-use Jinja** — readability matters more than cleverness.
- **Don't make everything incremental** — only when table rebuilds are too slow.

---

**Next step:** Hands-on — build a dbt project from scratch with DuckDB
