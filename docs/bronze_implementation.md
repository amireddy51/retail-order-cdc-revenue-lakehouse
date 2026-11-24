
# Phase 2 — Bronze Layer Implementation

## 1. Objective

Implement a reliable and replay-safe Bronze ingestion
pipeline for the Retail Order CDC & Revenue Lakehouse
using Databricks Free Edition, PySpark, Unity Catalog
and Delta Lake.

## 2. Environment

- Platform: Databricks Free Edition
- Compute: Serverless
- Spark version: 4.2.0
- Catalog: retail_poc2
- Schema: bronze
- Storage format: Delta Lake
- Table type: Unity Catalog managed tables
- Validation date: 2026-11-24

## 3. Source Data

| Dataset | Source records |
|---|---:|
| Customers | 100 |
| Products | 30 |
| Initial orders | 500 |
| Order CDC events | 15 |
| Total | 645 |

Source files are stored in a Unity Catalog Volume.

The CDC dataset consists of three batches containing
five events each.

## 4. Bronze Target Tables

- retail_poc2.bronze.customers_raw
- retail_poc2.bronze.products_raw
- retail_poc2.bronze.orders_raw
- retail_poc2.bronze.orders_cdc_raw

## 5. Implementation Design

The ingestion pipeline uses:

- Explicit PySpark schemas
- DecimalType(18,2) for monetary values
- LongType for sequence numbers
- CSV header validation
- Financial precision validation
- FAILFAST parsing
- Reusable ingestion functions
- Insert-only Delta MERGE
- Source-to-target reconciliation

## 6. Ingestion Metadata

| Column | Purpose |
|---|---|
| _source_file | Source file path |
| _source_file_name | Source file name |
| _ingestion_timestamp | Ingestion processing time |
| _record_hash | SHA-256 hash of parsed business values |

## 7. Idempotency Strategy

The ingestion identity consists of:

- Source file path
- Record hash

An insert-only Delta MERGE inserts records only when
their ingestion identity does not already exist.

Existing Bronze records are not updated or deleted
by the ingestion function.

## 8. Initial Ingestion Results

| Table | Source | Inserted | Target |
|---|---:|---:|---:|
| customers_raw | 100 | 100 | 100 |
| products_raw | 30 | 30 | 30 |
| orders_raw | 500 | 500 | 500 |
| orders_cdc_raw | 15 | 15 | 15 |
| Total | 645 | 645 | 645 |

Result: PASS

## 9. Data Reconciliation Results

| Validation | Result |
|---|---|
| Source-to-target record counts | PASS |
| Full parsed-record reconciliation | PASS |
| Missing records | 0 |
| Unexpected records | 0 |
| Duplicate ingestion identity groups | 0 |
| Invalid metadata records | 0 |
| Delta storage format | PASS |
| Managed table type | PASS |

Tables validated: 4
Passed: 4
Failed: 0

## 10. Idempotency Test Results

The same source datasets were processed again.

Delta Time Travel was used to compare the original
version 1 snapshots against the latest table state.

| Table | Before | After | Inserted |
|---|---:|---:|---:|
| customers_raw | 100 | 100 | 0 |
| products_raw | 30 | 30 | 0 |
| orders_raw | 500 | 500 | 0 |
| orders_cdc_raw | 15 | 15 | 0 |
| Total | 645 | 645 | 0 |

Removed or changed records: 0
Added or changed records: 0

Tables tested: 4
Passed: 4
Failed: 0

Result: PASS

## 11. Issue Encountered and Resolution

The initial table creation failed because the
errorifexists write mode was unsupported in the
current Serverless Spark Connect environment.

Resolution:

Changed the empty-table initialization write mode
from errorifexists to ignore.

The insert-only Delta MERGE strategy remained
unchanged.

All four tables were subsequently created and
populated successfully.

## 12. Design Assumptions and Limitations

- Source files are assumed to be immutable.
- Pipeline execution is assumed to be sequential.
- Identical ingestion identities within the same
  source dataset cause ingestion to fail.
- Business-level duplicate handling belongs to Silver.
- Bronze preserves parsed values, not original CSV bytes.
- Each Delta table write is independently transactional.
- Cross-table atomicity is not implemented.
- Concurrent execution and partial-failure recovery
  have not yet been tested.
- Record counts are specific to the current POC dataset.
- Persistent audit logging and orchestration are pending.

## 13. Final Status

Bronze ingestion: PASS
Data reconciliation: PASS
Sequential idempotency testing: PASS

The Bronze implementation is validated for the
current POC dataset.

The next phase will implement Silver data quality,
CDC ordering, upserts, soft deletes and controlled
handling of invalid and late-arriving events.

