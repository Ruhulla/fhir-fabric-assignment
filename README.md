# FHIR Data Ingestion & Analytics Pipeline

## Overview
This project ingests Patient, Encounter, Observation, and Condition resources
from the public HAPI FHIR API (https://hapi.fhir.org/baseR4) and builds a
Medallion Architecture (Raw → Bronze → Silver → Gold) in Microsoft Fabric.

## Architecture

FHIR API
   ↓ (paginated fetch, incremental via _lastUpdated)
Raw Layer (Files/raw/{resource}/{date}/*.json)
   ↓ (flatten nested JSON)
Bronze Layer (Delta tables, one per resource)
   ↓ (clean, dedupe, SCD Type 2 history tracking)
Silver Layer (Delta tables with is_current/valid_from/valid_to)
   ↓ (join, denormalize for reporting)
Gold Layer (Delta tables ready for Power BI / reporting)

## Tables

| Table | Layer | Source | Primary Key | Notes |
|---|---|---|---|---|
| raw/Patient, raw/Encounter, raw/Observation, raw/Condition | Raw | FHIR API | - | JSON files, dated folders, includes extraction_timestamp + api_url_or_params metadata |
| bronze_patient | Bronze | Patient | patient_id | flattened, includes full_resource_json |
| bronze_encounter | Bronze | Encounter | encounter_id | flattened |
| bronze_observation | Bronze | Observation | observation_id | flattened |
| bronze_condition | Bronze | Condition | condition_id | flattened |
| silver_patient | Silver | bronze_patient | patient_id | SCD Type 2 (is_current, valid_from, valid_to) |
| silver_encounter | Silver | bronze_encounter | encounter_id | SCD Type 2 |
| silver_observation | Silver | bronze_observation | observation_id | SCD Type 2 |
| silver_condition | Silver | bronze_condition | condition_id | SCD Type 2 |
| gold_patient_encounter_summary | Gold | silver_patient + silver_encounter | - | joined reporting view |
| gold_patient_condition_summary | Gold | silver_patient + silver_condition | - | joined reporting view |
| gold_encounter_observation_summary | Gold | silver_encounter + silver_observation | - | joined reporting view |



## Modular Design

Instead of writing separate ingestion/transformation code per resource, this
project uses a single reusable set of functions (`fetch_all_pages`,
`save_raw_pages`, `scd2_merge`) applied to all four FHIR resources. Field
mappings per resource are the only resource-specific code, keeping the
pipeline logic itself fully reusable and free of hardcoding.

## How to Run

1. Open the notebook in Microsoft Fabric, attached to the project Lakehouse.
2. Run all cells top to bottom. This will:
   - Fetch paginated data for Patient, Encounter, Observation, Condition
     from the FHIR API (bounded to 5 pages per resource for demo purposes)
   - Save raw JSON to the Lakehouse Files area under `raw/{resource}/{date}/`
   - Convert raw JSON into flattened Bronze Delta tables
   - Clean and apply SCD Type 2 merge logic to produce Silver Delta tables
   - Join Silver tables into three Gold reporting tables
3. To simulate a new incremental load ("day 2"), simply re-run the notebook.
   The pipeline is idempotent — re-running with unchanged source data will
   not create duplicate Silver history.

## Incremental Loading

Incremental filtering is supported via the FHIR `_lastUpdated` search
parameter (e.g. `_lastUpdated=ge2026-07-18`), allowing the pipeline to pull
only records updated on or after a given date. Data was also sorted by
`-_lastUpdated` to ensure the most recently changed records are captured
first, since the public test server's dataset changes constantly.

## SCD Type 2 Verification

The pipeline was run twice in succession (simulating two incremental loads).
On the second run, the SCD2 merge correctly identified that no tracked
fields had changed for any record and inserted 0 new rows — confirming the
pipeline is idempotent and does not create duplicate/fake history when
source data is unchanged. Row counts remained consistent
(250 patients / 250 unique patient_ids) across both runs, verifying no
duplication occurred.

To verify SCD2 further, an update to any tracked field for an existing
record will cause the Silver merge to expire the old row
(`is_current = false`, `valid_to` populated) and insert a new current
version (`is_current = true`) — this logic is implemented in the
`scd2_merge` function and applied identically across all four resources.

## Orchestration

A Fabric Data Pipeline (`fhir_ingestion_pipeline`) was configured with a
Notebook activity to orchestrate the end-to-end run. Execution was
constrained during testing by Fabric trial-capacity limits
(HTTP 430 – TooManyRequestsForCapacity), a known limitation of the free
trial SKU under shared load, not a defect in the pipeline design. The
underlying notebook executes successfully end-to-end when run interactively,
and the pipeline activity is correctly configured to trigger it.

## Known Limitations

- The public HAPI FHIR test server (https://hapi.fhir.org/baseR4) is a
  shared sandbox with constantly changing data, contributed by many
  external users/testers. Results are not deterministic across runs.
- Ingestion is bounded to 5 pages per resource (250 records) per run for
  demo purposes; this limit is configurable via the `max_pages` parameter.
- Not all Observation records include `valueQuantity` — some use
  `valueString` or `valueCodeableConcept` instead, resulting in nulls for
  `value`/`unit` on those rows.
- `Encounter.period` was not consistently populated on this test server and
  was therefore excluded from the Bronze schema.

## Optional Extensions Not Completed

- XML format ingestion (JSON was used instead)
- Power BI report on Gold layer
