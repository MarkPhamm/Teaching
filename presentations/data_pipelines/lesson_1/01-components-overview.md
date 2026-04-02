---
marp: true
theme: default
paginate: true
---

# Data Pipeline Components

An end-to-end overview of modern data infrastructure

---

# What is a Data Pipeline?

A system that moves data from **point A** to **point B**, transforming it along the way so it becomes useful for decision-making.

**Key question:** How do we get raw, messy data from many sources into a clean, reliable place where people can analyze it?

---

# The Big Picture

```text
Sources → Staging → Storage → Transformation → Consumption
                        ↑                           
                  Orchestration                     
                  CI/CD + Containers                
```

Every component serves a purpose. Skip one, and the pipeline breaks — or worse, silently produces wrong data.

---

# 1. Sources

Where data originates. You don't control these — you adapt to them.

---

# 1.1 OLTP Systems (Operational Databases)

- The databases behind your applications (PostgreSQL, MySQL, SQL Server)
- Optimized for **writes** — fast inserts, updates, deletes
- Not designed for analytical queries (slow aggregations, expensive joins)

**Why we extract from them:** we need the data, but querying OLTP directly would slow down the application.

**Common extraction methods:**
- Full load — dump the entire table periodically
- Incremental load — only pull new/changed rows since last run
- CDC (Change Data Capture) — stream every insert/update/delete in real time

---

# 1.2 APIs

- REST APIs, GraphQL endpoints from SaaS tools
- Examples: Stripe (payments), Salesforce (CRM), Google Analytics, social media platforms

**Challenges:**
- Rate limits — you can only call so many times per minute
- Pagination — data comes in pages, you need to loop through all of them
- Schema changes — the API provider can change the response format without warning
- Authentication — tokens expire, keys rotate

---

# 1.3 Third-Party Sources

- Vendor data feeds (CSV/JSON drops to SFTP or S3)
- Partner data sharing (Snowflake data shares, data marketplaces)
- Government or public datasets

**Challenges:**
- No control over format, quality, or delivery schedule
- Need to validate every delivery — don't trust external data blindly

---

# 1.4 Web Scraping

- Extracting data from websites when no API exists
- Legal and ethical considerations — always check terms of service

**When to use:** last resort when structured data sources are unavailable.

---

# 2. Staging Area

The first landing zone for raw data. A buffer between sources and your analytical storage.

---

# 2.1 Why Stage?

- **Decouple extraction from transformation** — extract fast, transform later
- **Preserve raw data** — always keep an untouched copy (you'll need it when something breaks)
- **Replayability** — if transformation logic changes, reprocess from raw

**Principle:** extract data as-is first. Never transform during extraction — you lose the original.

---

# 2.2 Where to Stage

- **Cloud object storage** — S3 (AWS), GCS (GCP), ADLS (Azure)
- Cheap, durable, scalable
- Store in partitioned folders by date/source for easy management

**Typical zone pattern:**
- `raw/` — untouched data as extracted
- `cleaned/` — validated and lightly processed
- `curated/` — modeled and ready for consumption

---

# 3. Storage — Where Analytics Happens

The core of your data platform. Two main approaches.

---

# 3.1 Data Warehouse

A managed analytical database optimized for **reads** — fast aggregations, joins, window functions.

**Examples:** Snowflake, Redshift, BigQuery, Azure Synapse

**Pros:**
- SQL-native, familiar to analysts
- Managed infrastructure — less operational burden
- Built-in security, governance, scaling

**Cons:**
- Vendor lock-in
- Cost scales with compute usage
- Less flexible for non-SQL workloads (ML, unstructured data)

---

# 3.2 Lakehouse

An open architecture that combines the flexibility of a data lake with the structure of a warehouse.

**Components:**
- **Object storage** (S3) — stores the actual data files
- **Table format** (Apache Iceberg, Delta Lake) — adds ACID transactions, schema evolution, time travel
- **Query engine** (Trino, Spark) — runs SQL over the files
- **Catalog** (PostgreSQL, Hive Metastore, AWS Glue) — tracks where tables and partitions live

**Pros:**
- Open formats — no vendor lock-in
- Full control over infrastructure
- Supports SQL, ML, and streaming workloads

**Cons:**
- More operational overhead
- Need to manage more moving parts
- Requires stronger engineering skills

---

# 3.3 Warehouse vs Lakehouse — When to Choose What

| Factor | Warehouse | Lakehouse |
|---|---|---|
| Team skill | SQL-heavy, analyst-driven | Engineering-heavy |
| Data types | Structured | Structured + unstructured |
| Control needs | Low — prefer managed | High — need open formats |
| Budget model | Pay per query/compute | Pay for infra you manage |
| Speed to value | Faster to set up | More upfront investment |

There is no universally "better" choice — it depends on the team, data, and business.

---

# 4. Orchestration

The brain of the pipeline. Decides **what** runs, **when**, and **what happens if it fails**.

---

# 4.1 What Orchestration Does

- Schedule and trigger pipeline tasks
- Manage dependencies between tasks (task B waits for task A)
- Handle retries and failure alerts
- Provide visibility — logs, run history, lineage

**Without orchestration:** you're running cron jobs and praying.

---

# 4.2 Tools

- **Apache Airflow** — the industry standard, DAG-based, Python-native. Large community, steep learning curve.
- **Dagster** — asset-based (think about *what* you're producing, not *how* to run it). Better developer experience, newer.
- **Prefect** — flow-based, Pythonic, less boilerplate than Airflow.
- **Mage** — newer, visual + code hybrid, good for smaller teams.

**Key decision:** DAG-based (define task order) vs asset-based (define what outputs you want). Asset-based is the trend.

---

# 5. CI/CD and Containerization

How you ship and run pipeline code reliably.

---

# 5.1 Why It Matters

- Pipelines are software — they need version control, testing, and deployment processes
- "It works on my laptop" is not acceptable for data that drives business decisions

---

# 5.2 Docker / Containers

- Package your pipeline code + dependencies into a reproducible image
- Same environment in dev, staging, and production
- No more "but I have a different Python version" issues

**Key idea:** the container is the unit of deployment. Build once, run anywhere.

---

# 5.3 CI/CD

- **CI (Continuous Integration)** — automatically test pipeline code on every push (linting, unit tests, dbt tests)
- **CD (Continuous Deployment)** — automatically deploy to production after tests pass

**Tools:** GitHub Actions, GitLab CI, Jenkins

**For dbt specifically:** test models in CI before merging → catch broken transformations before they hit production data.

---

# 5.4 Infrastructure as Code

- **Terraform** — define cloud resources (S3 buckets, databases, IAM roles) in code
- Reproducible, reviewable, version-controlled infrastructure
- No more "who clicked what in the AWS console?"

---

# 6. Consumption — BI Tools and BI as Code

Where the data finally becomes useful to the business.

---

# 6.1 Traditional BI Tools

- **Tableau, Power BI, Looker, Mode** — drag-and-drop dashboards, reports, explorations
- Accessible to non-technical users
- Connect directly to the warehouse

**Risk:** dashboard sprawl — hundreds of dashboards, nobody knows which is the source of truth.

---

# 6.2 BI as Code

Treat metrics and dashboards like software — version-controlled, tested, reviewed.

**Approaches:**
- **Metrics layer** (dbt metrics, Cube.js) — define business metrics once, use everywhere
- **Notebooks** (Jupyter, Hex) — code-driven analysis, reproducible
- **Streamlit / Evidence** — build data apps and reports from code

**Why it matters:** when a metric definition lives in 5 different dashboards and they all show different numbers, trust in data dies.

---

# Putting It All Together

```text
  APIs ──┐                                              ┌── Tableau
  OLTP ──┤    ┌─────────┐   ┌───────────┐   ┌─────┐    ├── Power BI
  3rd  ──┼───▶│ S3 (raw)│──▶│ Snowflake │──▶│ dbt │───▶├── Notebooks
  Web  ──┘    └─────────┘   └───────────┘   └─────┘    └── Streamlit
                     ▲                          ▲
                     └──── Airflow / Dagster ────┘
                     └──── Docker + CI/CD ──────┘
```

---

# Key Takeaways

1. **Sources are messy** — design for failure, validate everything
2. **Always stage raw data** — you will need to reprocess
3. **Warehouse vs lakehouse** — depends on team, not hype
4. **Orchestration is non-negotiable** — without it, pipelines are just scripts
5. **Treat pipelines like software** — CI/CD, containers, version control
6. **BI is only as good as the pipeline behind it** — garbage in, garbage out

---

# Questions?
