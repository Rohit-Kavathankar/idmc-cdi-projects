# Shipping & Logistics Analytics

Four IDMC CDI mappings that enrich orders, detect shipment delays, reconcile inventory and consolidate everything into a KPI summary.

## Business problem

A supply-chain team wants to know:

- which orders are high priority and when they should arrive
- which shipments are late, and by how much
- which items are below their reorder level
- one consolidated KPI view across orders and delays

## Data flow

```
orders.csv + vendors.csv ──► order_enrichment ──► sh_enriched_orders.csv ──┐
                                                                            ├──► m_kpi_consolidation ──► sh_kpi_summary.csv
shipments.csv ──► m_shipment_delay_detection ──► sh_delay_report.csv ──────┘

inventory.csv ──► m_inventory_reconciliation ──┬──► sh_inventory_low_stock.csv
                                                └──► sh_inventory_ok.csv
```

All sources and targets are CSV files on AWS S3.

## Mappings

### 1. `order_enrichment`
- **Sources:** orders, vendors
- **Logic:**
  - Joiner links each order to its vendor
  - Expression sets `order_priority` (`HIGH` if quantity > 10, else `NORMAL`)
  - Expression also derives an expected-delivery value for open orders: order date plus vendor lead time (`ADD_TO_DATE`), while delivered or cancelled orders keep their status
- **Target:** `sh_enriched_orders.csv`
- **Transformations:** Source ×2, Joiner, Expression, Target

### 2. `m_shipment_delay_detection`
- **Source:** shipments
- **Logic:**
  - Sorter orders the shipments
  - Expression calculates `delay_days` (`DATE_DIFF` of actual vs expected delivery), an `is_delayed` flag (`YES`/`NO`) and a delay category: **On Time** (0 days), **Minor** (1–3 days), **Major** (more than 3 days)
- **Target:** `sh_delay_report.csv`
- **Transformations:** Source, Sorter, Expression, Target

### 3. `m_inventory_reconciliation`
- **Source:** inventory
- **Logic:**
  - Aggregator totals stock quantity (`SUM(stock_qty)`)
  - Expression compares total stock to the reorder level: `LOW` or `OK`, plus the gap (`total_stock - reorder_level`)
  - Router sends items to separate low-stock and OK-stock targets
- **Targets:** low-stock list, OK-stock list
- **Transformations:** Source, Aggregator, Expression, Router, Target ×2

### 4. `m_kpi_consolidation`
- **Sources:** enriched orders, delay report
- **Logic:**
  - Joiner combines the two datasets
  - Expression cleans nulls (quantity and delay days default to 0) and creates `on_time_flag` and a high-priority flag
  - Aggregator calculates order count, on-time count, total quantity and average delay days
  - Sequence Generator (`NEXTVAL`) assigns a unique KPI row ID
- **Target:** `sh_kpi_summary.csv`
- **Transformations:** Source ×2, Joiner, Expression, Aggregator, Sequence Generator, Target

## Skills demonstrated

Joiner · Sorter · Aggregator · Router · Sequence Generator · date arithmetic · null handling · derived flags and KPIs · multi-stage design where two mappings feed a consolidation mapping