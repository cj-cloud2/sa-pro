**Puzzle 1: Data Lake Foundation Challenge**

**Scenario:** AnalyticsCorp is building a petabyte-scale data lake in S3 for IoT sensor data (raw streams), customer events (curated Parquet), and ML features (enriched). Crisis: Analysts report "schema drift" (new IoT fields breaking Athena queries), duplicate datasets across teams, and undiscoverable raw data inflating costs. Goals: (1) Organize zones with metadata, (2) Handle evolving schemas, (3) Ensure governance without reprocessing.

**Zone Mapping + Crisis Resolution Grid:**


| Challenge | Primary Catalog Strategy | Key Feature for Crisis | Why Over Other Approaches? |
| :-- | :-- | :-- | :-- |
| 1. **Raw IoT Streams (Zone 1)** | ? | *Hint: Auto-discover schema from JSON/Avro without manual DDL.* | ? |
| 2. **Curated Events (Zone 2)** | ? | *Hint: Evolve table definitions for Parquet field additions.* | ? |
| 3. **Data Duplication Across Teams** | ? | *Hint: Centralized metadata prevents S3 sprawl/scans.* | ? |
| 4. **Query Optimization** | ? | *Hint: Partition metadata enables Athena partition pruning.* | ? |

**Your Task (10-15 mins):** Fill the grid. For each, select **one Glue Data Catalog feature**, its crisis-solving benefit, and exclude 1 alternative (e.g., manual manifests). Bonus: Sketch zone flow (Raw → Curated). Discuss: How does Catalog reduce costs vs. S3-only?

**Submit:** Completed grid + 1-sentence "crisis fix" per row.

***

### Solution Key with Detailed Justifications

#### Challenge 1: Raw IoT Streams (Zone 1) → **Glue Crawlers**

**Teaching from Scratch:** A data lake organizes data into *zones*—Raw (Landing Zone: immutable, schema-on-read), Curated (Refined: cleaned/transformed), and Consumption (Optimized: partitioned/aggregated). AWS Glue Data Catalog acts as the *central metadata repository* (Hive-compatible), storing table definitions, schemas, partitions, and serdes across S3/RDS. Crawlers are automated *schema discovery agents*—deploy once, point to S3 paths (e.g., JSON/Avro streams), infer schema/partition keys (e.g., date/hour), and populate Catalog tables. Schedule daily; handles nested/semi-structured data without ETL.

**Why Best Here:** Auto-creates Raw zone tables from IoT streams—no manual schema for volatile data; enables immediate Athena discovery.
**Action Plan:** Create crawler → Target s3://raw/iot/ → Run → Query via Athena.
**Why Not Others?** Manual DDL fails on drift; table versioning is for post-curation.
**Crisis Fix:** Eliminates "undiscoverable raw data" by indexing automatically.

#### Challenge 2: Curated Events (Zone 2) → **Table Versioning \& Schema Evolution**

**Teaching from Scratch:** Data evolves—new fields added (e.g., Parquet appends "user_agent"). Glue Catalog supports *schema evolution* via versioning: Each crawler run creates a new table version with updated schema, preserving history. Athena queries latest version by default but supports @version tags. Backward-compatible changes (added columns) auto-merge; breaking changes flag warnings. Integrates with Glue ETL jobs for transformations (e.g., PySpark cleans to Parquet).

**Why Best Here:** Handles "schema drift" in customer events—new IoT fields append without reprocessing historical Parquet.
**Action Plan:** Crawl curated/ → Enable versioning → Athena SELECT * FROM events VERSION 5.
**Why Not Others?** Crawlers alone don't version; partitions optimize location, not schema.
**Crisis Fix:** Keeps queries running despite field additions, avoiding ETL rework.

#### Challenge 3: Data Duplication Across Teams → **Centralized Catalog Governance**

**Teaching from Scratch:** Without Catalog, teams duplicate S3 copies or scan entire buckets (costly). Catalog provides *unified discovery*—search tables by name/tags, enforce naming (e.g., dept=analytics), and integrate Lake Formation for permissions (column-level). One metadata entry per logical dataset; multiple teams query same S3 path. Tags/partitions link to S3 lifecycle policies (e.g., Glacier raw after 90 days).

**Why Best Here:** Single source-of-truth prevents "duplicate datasets"; teams find/reuse via Glue/Athena consoles.
**Action Plan:** Tag tables (e.g., owner=marketing) → Lake Formation grants → Share cross-account.
**Why Not Others?** S3 prefixes alone lack semantics/search; crawlers populate but don't govern.
**Crisis Fix:** Stops sprawl, cuts storage costs 30-50% via deduplication.

#### Challenge 4: Query Optimization → **Partition Projection \& Metadata**

**Teaching from Scratch:** Athena scans billed by data scanned—*partition metadata* in Catalog tells it to prune (e.g., WHERE date='2026-01-15' skips 99%). Glue infers partitions (year/month/day) during crawl or via ETL (MSCK REPAIR). Projection pushes static partitions (e.g., region=us-east-1) to Catalog, avoiding S3 LIST ops for dynamic queries.

**Why Best Here:** Partition metadata on all zones drops scan costs from PB to TB; essential for petabyte lakes.
**Action Plan:** Crawl with --partitions date/hour → Athena auto-prunes.
**Why Not Others?** Versioning/schema handle structure, not location optimization.
**Crisis Fix:** Fixes slow/expensive "breaking queries" via pruning.
