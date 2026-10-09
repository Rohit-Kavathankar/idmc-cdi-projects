# Healthcare Claims Pipeline

A four-mapping IDMC CDI pipeline that prepares patient, claims and appointment data for reporting, orchestrated by a single taskflow with status checks and failure notifications.

## Business problem

A healthcare provider has raw patient, claims, provider, insurance-plan and appointment files. Analysts need:

- a clean master list of active patients with age and insurance details
- claims enriched with provider information, with bad claims separated out
- monthly claims totals per patient
- appointment counts and no-show rates per patient

## Data flow

```
patients.csv + insurance_plans.csv ──► m_patient_master ──► hc_patient_master.csv ─┐
                                                                                    ├──► m_appointment_summary ──► hc_appt_summary
appointments.csv ───────────────────────────────────────────────────────────────────┘

claims.csv + providers.csv ──► m_claims_enrichment ──┬──► hc_enriched_claims ──► m_claims_aggregation ──► hc_claims_summary.csv
                                                      └──► hc_rejected_claims
```

All sources and targets are CSV files on AWS S3, accessed through an IDMC S3 connection.

## Mappings

### 1. `m_patient_master`
- **Sources:** patients, insurance plans (Lookup)
- **Logic:**
  - Filter keeps `ACTIVE` patients
  - Lookup `lkp_insurance_plans` brings in plan details
  - Expression derives `age` from date of birth (`TRUNC(DATE_DIFF(SYSDATE, TO_DATE(dob,'YYYY-MM-DD'),'MM')/12)`) and an active flag (`IIF(status='ACTIVE','Y','N')`)
- **Target:** `hc_patient_master.csv`
- **Transformations:** Source, Filter, Lookup, Expression, Target

### 2. `m_claims_enrichment`
- **Sources:** claims, providers
- **Logic:**
  - Joiner combines claims with provider details on provider ID
  - Expression builds `provider_label` (`provider_name || '-' || specialty`) and `claim_month` (`Mon-YYYY` from the claim date)
  - Router separates valid claims from rejected claims
- **Targets:** enriched claims, rejected claims
- **Transformations:** Source ×2, Joiner, Expression, Router, Target ×2

### 3. `m_claims_aggregation`
- **Source:** enriched claims
- **Logic:**
  - Sorter orders the data for grouping
  - Aggregator calculates `COUNT(claim_id)` and `SUM(amount)` per patient per month
  - Expression builds a readable summary key (`'Patient' || patient_id || ' | ' || claim_month`)
- **Target:** `hc_claims_summary.csv`
- **Transformations:** Source, Sorter, Aggregator, Expression, Target

### 4. `m_appointment_summary`
- **Sources:** appointments, patient master (output of mapping 1)
- **Logic:**
  - Joiner links appointments to patients
  - Filter drops rows with a missing no-show flag
  - Aggregator calculates `COUNT(appt_id)` as total visits and `SUM(no_show_flag)` as total no-shows
  - Expression derives the no-show rate: `IIF(total_visits = 0, 0, ROUND((total_noshows / total_visits) * 100, 2))`
- **Target:** `hc_appt_summary`
- **Transformations:** Source ×2, Joiner, Filter, Aggregator, Expression, Target

## Orchestration: `tf_healthcare_pipeline`

A taskflow runs the four mapping tasks (`mt_patient_master`, `mt_claims_enrichment`, `mt_claims_aggregation`, `mt_appointment_summary`) in dependency order.

- Each mapping task is configured to wait for completion
- **Decision** steps check the `Task_Status` output of the preceding mapping task
- **Notification** tasks send an email if a step fails, so the pipeline never fails silently

## Skills demonstrated

Lookup enrichment · Joiner · Filter · Aggregator · Sorter · Router-based reject handling · date and string functions · taskflow orchestration with Decision and Notification steps · chaining mappings so one's output feeds the next