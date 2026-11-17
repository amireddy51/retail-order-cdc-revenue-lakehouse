# Retail Order CDC & Revenue Lakehouse

## 1. Project Objective
Build a production-style retail data pipeline using Databricks Free Edition, PySpark, Delta Lake, SQL, and GitHub.

The pipeline processes initial retail data, applies order changes using Change Data Capture (CDC), validates data quality, and produces revenue analytics.

## 2. Architecture
Retail CSV Files -> Unity Catalog Volume -> Bronze Delta Tables -> Silver Delta Tables -> Gold Revenue Tables

Audit tables record pipeline executions, rejected records, and data quality results.

## 3. Databricks Resources
Catalog: retail_poc2
Schemas: landing, bronze, silver, gold, audit
Source Volume: retail_poc2.landing.source_files
Source path: /Volumes/retail_poc2/landing/source_files/

Source directories:
- initial/
- cdc/
- test_cases/

## 4. Source Data
- customers.csv: customer information
- products.csv: product information
- orders.csv: initial retail orders
- orders_cdc_batch_*.csv: inserts, updates, and deletes to retail orders
- test_cases/: duplicate events, invalid orders, and late-arriving events

Exact column definitions and data types must be confirmed against the dataset specification before implementation.

## 5. Bronze Layer
Read source CSV files with explicit schemas. Preserve original records and CDC events in Delta tables. Add ingestion metadata, including source filename, batch identifier, and ingestion timestamp. Do not silently discard invalid records.

## 6. Silver Layer
Validate identifiers, required fields, data types, order references, and business rules. Separate rejected records. Deduplicate CDC events and apply changes in deterministic event order using Delta MERGE. Make retries and reprocessing idempotent.

## 7. CDC Rules
I = Insert a new order.
U = Update an existing order.
D = Delete an order from the current active order state.

Use event ordering and duplicate handling so repeated batches do not change the final result. Late-arriving events must not overwrite newer order states.

## 8. Gold Layer
Create curated revenue analytics from the validated current order state. Revenue calculations must account for quantity, unit price, order status, cancellations, and deletes according to the finalized business rules. Avoid double counting after CDC updates or reprocessing.

## 9. Data Quality and Testing
Test source schemas, null identifiers, duplicates, invalid values, CDC inserts/updates/deletes, late events, referential integrity, revenue reconciliation, and idempotent reruns.

## 10. Audit and Monitoring
Capture pipeline run IDs, processing times, batch identifiers, row counts, rejected records, test results, and failure messages in the audit schema.

## 11. Orchestration
Use Databricks Jobs to execute the notebooks in dependency order, with validation before downstream processing. Verify Free Edition capabilities before configuring Jobs.

## 12. GitHub and Delivery
Maintain documentation, complete versioned notebooks, SQL, synthetic sample data, tests, and configuration in the separate retail-order-cdc-revenue-lakehouse repository.

Milestones:
1. Architecture and dependency validation
2. Bronze ingestion
3. Silver validation and CDC
4. Gold revenue analytics
5. Automated testing and audit
6. Jobs orchestration and final documentation

## 13. Implementation Gate
Do not begin pipeline implementation until source schemas, CDC ordering rules, revenue calculations, environment dependencies, and acceptance tests have been reviewed and approved.

POC-1 resources must remain untouched.
