# Patient Vital Monitoring — Real-Time Data Pipeline on GCP

An end-to-end, fully automated real-time data engineering project built on Google Cloud 
Platform. Simulated patient vitals (heart rate, SpO2, temperature, blood pressure) are 
streamed through Pub/Sub, processed with Apache Beam on Dataflow through a Bronze → 
Silver → Gold medallion architecture, and visualized live in Power BI via DirectQuery.

## Architecture

Simulator (Python) → Pub/Sub → Dataflow (Apache Beam) → GCS (Bronze/Silver) → BigQuery (Gold) → Power BI (DirectQuery)

## Tech Stack
- Python
- Google Cloud Pub/Sub
- Google Cloud Dataflow (Apache Beam)
- Google Cloud Storage
- BigQuery
- Power BI

## How It Works

**1. Data Source (Simulator)**
`patient_vitals_simulator.py` generates one record every 2 seconds for 20 simulated 
patients — heart rate, SpO2, temperature, and blood pressure (systolic/diastolic) — 
and publishes each as JSON to a Pub/Sub topic. A configurable `ERROR_RATE` randomly 
injects missing fields, negative values, or out-of-range values to simulate real-world 
data quality issues.

**2. Bronze Layer**
The Dataflow pipeline reads raw messages from the Pub/Sub subscription, decodes them, 
and writes the unmodified JSON directly to a `bronze/` folder in GCS every 60-second window.

**3. Silver Layer**
Records are parsed, validated (removing nulls, negative heart rates, out-of-range SpO2 
readings), and enriched with a calculated `risk_score` and `risk_level` (Low / Moderate / 
High) based on heart rate, temperature, and SpO2. Cleaned records are written to `silver/` in GCS.

**4. Gold Layer**
Silver records are grouped by `patient_id` and aggregated (average heart rate, SpO2, 
temperature, and the max risk level observed) before being appended to a BigQuery table.

**5. Visualization**
Power BI connects to the BigQuery Gold table using **DirectQuery** , so 
dashboards refresh automatically as new data streams in.


