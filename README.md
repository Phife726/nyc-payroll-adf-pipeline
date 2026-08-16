# NYC Payroll Data Integration Pipelines

## Project Overview
This project implements an automated end-to-end data integration and ETL pipeline using Azure Data Factory (ADF), Azure Data Lake Storage Gen2 (ADLS Gen2), Azure SQL Database, and Azure Synapse Analytics. The solution ingests municipal employee master data and annual payroll transaction records, then prepares aggregated compensation metrics for analysis across city agencies and fiscal years.

The core objective is to transform raw payroll source files into curated, query-ready datasets that support agency-level reporting, workforce analysis, and compensation trend monitoring.

---

## Architecture and Data Flow

```text
ADLS Gen2 (raw CSV source files)
    │
    ▼
ADF Data Flows (ingest and normalize)
    │
    ▼
Azure SQL Database
    │   ├─ NYC_Payroll_AGENCY_MD
    │   ├─ NYC_Payroll_EMP_MD
    │   ├─ NYC_Payroll_TITLE_MD
    │   ├─ NYC_Payroll_Data_2020
    │   ├─ NYC_Payroll_Data_2021
    │   └─ NYC_Payroll_Summary
    │
    ▼
ADF Summary Data Flow
    ├─ Union payroll datasets
    ├─ Filter by fiscal year
    ├─ Derive TotalPaid = RegularGrossPaid + TotalOTPaid + TotalOtherPay
    └─ Aggregate by AgencyName and FiscalYear
    │
    ├───────────────┬──────────────────────────────┐
    │               │                              │
    ▼               ▼                              ▼
SQL Summary      ADLS Gen2 dirstaging           Azure Synapse Analytics
Table            staging files                   Serverless SQL external table
```

### Components
1. Azure Data Lake Storage Gen2
   - Stores raw source files in folders such as `dirpayrollfiles`, `dirhistoryfiles`, and `dirstaging`.
   - Acts as both the landing zone for source data and the staging destination for aggregated summary output.

2. Azure SQL Database
   - Hosts master dimension tables for agency, employee, and title metadata.
   - Stores yearly payroll fact data for 2020 and 2021.
   - Maintains the summary table used for downstream analytics.

3. Azure Data Factory
   - Orchestrates ingestion and transformation activities through parameterized pipeline execution.
   - Uses Mapping Data Flows for both master data and transactional payroll processing.

4. Azure Synapse Analytics
   - Queries staged summary files in ADLS Gen2 via a serverless SQL external table.
   - Provides a data exploration and analytical layer over the aggregated results.

---

## Data Pipeline Details

### Orchestration Pipeline: `pl_nyc_payroll`
The main pipeline coordinates the ingestion and transformation flow in the following order:

- Parallel master data ingestion:
  - `df_agency`
  - `df_emp`
  - `df_title`
- Transactional payroll ingestion after master data success:
  - `df_payroll2020`
  - `df_payroll2021`
- Aggregation and export through `df_summary`

### Summary Data Flow: `df_summary`
The summary transformation performs the following actions:

- Reads payroll records from the SQL tables for both 2020 and 2021.
- Normalizes column names between the data sources.
- Unions the two years into a single dataset.
- Filters records by fiscal year using a dynamic parameter.
- Derives the compensation metric:
  - `TotalPaid = RegularGrossPaid + TotalOTPaid + TotalOtherPay`
- Aggregates totals by `AgencyName` and `FiscalYear`.
- Writes results to both:
  - the SQL summary table: `NYC_Payroll_Summary`
  - the ADLS Gen2 staging directory: `dirstaging`

### Parameterization
The pipeline is designed to support configurable fiscal-year filtering:

- Pipeline-level parameter: `pipeline_param_fiscalyear`
- Data flow-level parameter: `dataflow_param_fiscalyear`
- Filter logic:
  - `toInteger(FiscalYear) >= $dataflow_param_fiscalyear`

This allows the summary data flow to process data starting from a selected fiscal year threshold.

---

## Synapse Serverless Implementation Note
This repository reflects the updated lab environment pattern where dedicated SQL pools are replaced by Synapse serverless SQL external tables. Summary data exported to `dirstaging/` is queried through an external data source and an external table definition, enabling analytics without a dedicated warehouse cluster.

---

## Repository Contents
- `factory/` – ADF factory configuration and global parameters
- `pipeline/` – Pipeline definitions such as `pl_nyc_payroll`
- `dataflow/` – Mapping Data Flow definitions, including summary aggregation logic
- `dataset/` – Source and sink dataset definitions for ADLS and SQL objects
- `linkedService/` – Linked service definitions for ADLS and SQL connectivity
- `screenshots/` – Execution evidence, verification screenshots, and pipeline outputs

---

## Verification and Proof of Work
All pipeline execution artifacts, successful activity runs, and validation screenshots are documented in the [screenshots/](./screenshots) directory. This provides evidence of the end-to-end operational flow and confirms the pipeline behavior against the source data and aggregation outputs.

---

## Summary
This solution demonstrates a practical Azure analytics pattern for ingesting public-sector payroll data, normalizing heterogeneous source files, creating reusable transformation logic in ADF, and exposing summarized results for analytical querying in Synapse. It is a complete example of ETL orchestration, data engineering, and serverless analytics in the Azure ecosystem.
