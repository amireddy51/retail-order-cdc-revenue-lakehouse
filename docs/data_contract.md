# Retail Order CDC & Revenue Lakehouse — Data Contract

## 1. Project Overview

This project implements a production-style retail data engineering pipeline using Databricks Free Edition, PySpark, Delta Lake, Unity Catalog, and GitHub.

The architecture follows the Medallion pattern:

**Landing → Bronze → Silver → Gold**

The objective is to ingest retail source data, process change data capture (CDC), enforce data quality, and generate accurate revenue analytics.

## 2. Source Data

**Unity Catalog:** `retail_poc2`

**Landing Volume:** `/Volumes/retail_poc2/landing/source_files`

| Source | Location | Validated records |
|---|---|---:|
| Customers | `initial/customers.csv` | 100 |
| Products | `initial/products.csv` | 30 |
| Orders | `initial/orders.csv` | 500 |
| CDC Batch 001 | `CDC/orders_cdc_batch_001.csv` | 5 |
| CDC Batch 002 | `CDC/orders_cdc_batch_002.csv` | 5 |
| CDC Batch 003 | `CDC/orders_cdc_batch_003.csv` | 5 |

The `CDC` directory name is case-sensitive.

## 3. Source Schemas

### Customers

| Column | Inferred type |
|---|---|
| customer_id | STRING |
| customer_name | STRING |
| email | STRING |
| city | STRING |
| state | STRING |
| created_at | TIMESTAMP |

### Products

| Column | Inferred type |
|---|---|
| product_id | STRING |
| product_name | STRING |
| category | STRING |
| unit_price | DOUBLE |
| active_flag | BOOLEAN |

### Initial Orders

| Column | Inferred type |
|---|---|
| order_id | STRING |
| customer_id | STRING |
| product_id | STRING |
| order_date | DATE |
| quantity | INTEGER |
| unit_price | DOUBLE |
| order_status | STRING |
| updated_at | TIMESTAMP |
| source_sequence | INTEGER |

### Order CDC Events

| Column | Inferred type |
|---|---|
| event_id | STRING |
| order_id | STRING |
| operation | STRING |
| event_sequence | INTEGER |
| event_timestamp | TIMESTAMP |
| customer_id | STRING |
| product_id | STRING |
| order_date | DATE |
| quantity | INTEGER |
| unit_price | DOUBLE |
| order_status | STRING |
| updated_at | TIMESTAMP |

These are the schemas observed during initial validation. Explicit ingestion schemas and detailed quality constraints will be defined before implementing Bronze and Silver.

## 4. CDC Operations

| Operation | Meaning | Intended behavior |
|---|---|---|
| I | Insert | Add a new order |
| U | Update | Update an existing order |
| D | Delete | Soft-delete the order |

Bronze preserves the incoming CDC events for traceability.

Silver maintains the latest valid order state.

## 5. CDC Ordering

Approved ordering priority:

1. `event_sequence` — primary ordering key.
2. `event_timestamp` — secondary ordering key.

Ordering is evaluated independently for each `order_id`.

### Proposed Conflict Policy

- Higher sequence: apply the newer event.
- Equal sequence with later timestamp: apply the later event.
- Identical replay: do not modify the current order state.
- Equal sequence and timestamp with conflicting payloads: quarantine for investigation.
- Older events: retain in Bronze without overwriting newer Silver state.

This policy is approved as a design direction but is not yet implemented or tested.

## 6. CDC Sequence Validation

Observed initial order sequence range: **1001–1500**.

Observed CDC event sequence range: **2001–2015**.

Validation findings:

- No CDC events matching initial orders had an equal or older sequence.
- No duplicate CDC event IDs were detected in the three regular CDC batches.
- No equal-sequence/timestamp collisions were detected.
- No null CDC ordering fields were detected.

These findings apply to the validated source files, not to all possible future inputs.

## 7. Soft-Delete Policy

Approved behavior:

- Preserve CDC delete events in Bronze.
- Retain deleted orders in Silver.
- Set `is_deleted = true`.
- Exclude deleted orders from Gold revenue calculations.
- Prevent older CDC events from accidentally reactivating deleted orders.

## 8. Revenue Business Rule

Only orders with `order_status = 'COMPLETED'` and `is_deleted = false` contribute to revenue.

**Order revenue = quantity × order unit_price**

The price stored on the order is used instead of the current product catalog price.

Orders that do not qualify contribute zero revenue.

Revenue calculations will be based on the latest valid Silver order state to avoid double counting CDC updates.

## 9. Data Quality and Testing

Planned validations include:

- Required fields and schema checks.
- Duplicate event detection.
- Invalid order value handling.
- CDC event ordering and replay idempotency.
- Late-arriving event protection.
- Soft-delete correctness.
- Revenue reconciliation.
- Quarantine and audit reporting.

Available test files:

| Test file | Validated records |
|---|---:|
| duplicate_events.csv | 2 |
| invalid_orders.csv | 2 |
| late_arriving_events.csv | 1 |

These files have been checked for readability; their expected failure scenarios still require dedicated tests.

## 10. Implementation Status

| Component | Status |
|---|---|
| Unity Catalog setup | Completed |
| Landing Volume and source uploads | Completed |
| Initial source validation | Passed |
| CDC ordering inspection | Passed |
| Revenue and delete design decisions | Approved |
| Bronze Delta ingestion | Not started |
| Silver CDC processing | Not started |
| Gold revenue models | Not started |
| Automated testing | Not started |

**Note:** This document describes the approved design and validated source characteristics. It does not claim that the end-to-end pipeline has already been implemented.

