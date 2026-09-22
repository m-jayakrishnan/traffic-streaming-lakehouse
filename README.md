# Traffic Streaming Lakehouse

An Azure Databricks  project that turns AI-camera JSON batches into confirmed intersection turning movements. It demonstrates a medallion flow: Auto Loader ingests raw events into Bronze, Silver validates and quarantines bad records, and Gold confirms complete entry-to-exit journeys.

## Architecture

`JSON batches → Bronze (Auto Loader/Delta) → Silver (validated + quarantine) → Gold (confirmed movements) → AI/BI dashboard`

The notebooks are intentionally separate so each layer can be run, inspected, and explained independently.

## Event schema

| Field | Meaning |
|---|---|
| `event_id` | Unique event key used by Silver MERGE |
| `batch_id` | Source batch identifier |
| `intersection_id` / `camera_id` | Intersection and camera context |
| `track_id` | Camera-local vehicle tracking identifier |
| `track_status` | `active` or `lost` |
| `event_time` | Event timestamp; never a date-only field |
| `vehicle_type` | `car`, `motorcycle`, `truck`, or `bus` |
| `detection_confidence` | Numeric confidence from 0.0 to 1.0 |
| `roi_zone` | Region-of-interest label, such as `north_entry` or `east_exit` |

## Data quality and reconciliation

Silver accepts required fields, valid status/type values, a non-null timestamp, and confidence in the inclusive range 0.0–1.0. Records missing `event_time`, `track_id`, or `detection_confidence` are quarantined with a reason; other invalid values are quarantined as `invalid_required_field_or_value`. A `lost` track is valid Silver data and is flagged `track_lost`.

The included `data/sample/batch_001.json` intentionally contains eight records: five Silver-valid events and three quarantined records. Therefore the expected batch reconciliation is **Bronze 8 = Silver 5 + Quarantine 3**. The sample also includes a lost track and an incomplete journey to exercise Gold filtering. `batch_002.json` shows a small cross-batch journey.

## Gold confirmation rules

Gold groups by `intersection_id`, `camera_id`, and `track_id`. It emits a movement only when the earliest zone ends in `_entry`, the latest zone ends in `_exit`, `exit_time > entry_time`, and no event for the track has `track_status = lost`. Incomplete journeys and lost tracks remain available in Silver but do not become confirmed movements.

## Dashboard

`Intersection Turning Movement Dashboard.lvdash.json` is an Azure Databricks AI/BI dashboard definition for exploring confirmed movements by camera, vehicle type, and movement.

## Repository structure

- `00_config.ipynb` — creates catalog, schemas, and volume using user-supplied workspace settings.
- `01_ingest-autoloader.ipynb` — reads `batch_*.json` with Auto Loader into Bronze.
- `02_Silver_Validation.ipynb` — validates, quarantines, deduplicates, and MERGEs events.
- `03_Gold_Turning_Movements.ipynb` — confirms complete turning movements and MERGEs Gold results.
- `data/sample/` — sanitized sample batches and their purpose.
- `Intersection Turning Movement Dashboard.lvdash.json` — dashboard definition.

## Azure Databricks prerequisites and run order

You need an Azure Databricks workspace with Unity Catalog, a SQL-capable cluster or serverless compute, permission to create the project catalog/schemas/volume, and a volume or cloud path containing the sample JSON files. Edit the clearly marked placeholders in `00_config.ipynb`; no credentials or storage identifiers belong in Git.

Run notebooks in this order: `00_config` → copy sample JSON into the configured landing path → `01_ingest-autoloader` → `02_Silver_Validation` → `03_Gold_Turning_Movements`. Confirm the Bronze/Silver/Quarantine counts before opening the dashboard.

## Known limitation

`track_id` is camera-local. This project does not perform cross-camera re-identification, so a vehicle changing cameras cannot be followed as one journey unless the source system provides a shared identifier.

This is an educational portfolio project, not a production-scale traffic management system.
