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
