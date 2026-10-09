# Banking Transaction Processing

Two IDMC CDI mappings that validate incoming bank transactions, separate rejects and AML-flagged records, and maintain an account dimension using an SCD Type 2 pattern.

## Business problem

A bank receives daily transaction files. Before analytics, it must:

- reject records with missing or invalid data and record the reason
- flag large transactions for anti-money-laundering (AML) review
- enrich valid transactions with account and branch details
- keep a history of changes to the account dimension rather than overwriting it

## Data flow

```
transactions.csv ──► m1_txn_ingest_validate ──┬──► bnk_valid_transaction.csv ──┐
                                               ├──► bnk_aml_flagged_transactions.csv
                                               └──► bnk_rejected_records.csv    │
                                                                                 ▼
accounts.csv + branches.csv + account_dimension.csv ──► Mapping6 (SCD Type 2) ──► enriched_transaction.csv
                                                                              └──► account_dimension (new / expired / new version / unchanged)
```

All sources and targets are CSV files on AWS S3.

## Mappings

### 1. `m1_txn_ingest_validate`
- **Source:** raw transactions
- **Logic:**
  - Expression sets an `error_reason` using nested `IIF` checks:
    - `NULL_ACCOUNT_ID`: account ID is null or blank (after `LTRIM`/`RTRIM`)
    - `NULL_AMOUNT`: amount is null
    - `INVALID_CHANNEL`: channel is not `Branch`, `ATM` or `Online`
  - Expression also sets `is_large_transaction` (`'Y'` when amount > 10,000), extracts the transaction hour (`TO_INTEGER(SUBSTR(transaction_time,1,2))`) and adds a load timestamp
  - Filters split records with and without an `error_reason`
  - Router splits clean records on `is_large_transaction`
- **Targets:** valid transactions, AML-flagged transactions, rejected records (with reason)
- **Transformations:** Source, Expression, Filter ×2, Router, Target ×3

### 2. `m2_txn_enrich_scd2`: enrichment and SCD Type 2
- **Sources:** valid transactions, accounts
- **Logic:**
  - Joiner `jnr_accounts` links transactions to accounts
  - Lookup `lkp_branches` adds branch details
  - Expression `exp_enrich_fields` builds fields such as full name (first name + last name) and date stamps
  - Lookup `lkp_existing_dim` fetches the current dimension row
  - Expression `exp_detect_change` compares incoming and existing values to classify each record
  - Router sends records to four outputs: **new**, **expired (old row closed)**, **new version**, **unchanged**
- **Targets:** enriched transactions plus the four dimension outputs
- **Transformations:** Source ×2, Joiner, Lookup ×2, Expression ×4, Router, Target ×5

#### SCD Type 2 approach
When an account's tracked attributes change, the old row is **expired** (end-dated) and a **new version** row is inserted, preserving history. Brand-new accounts are inserted as new, and unchanged accounts pass through untouched.

## Skills demonstrated

Data validation with reason codes · reject handling · conditional routing · multiple Lookups · SCD Type 2 · string and date functions · mapping outputs feeding downstream mappings