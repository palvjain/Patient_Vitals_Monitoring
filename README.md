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

### 1. Simulator (the data source)

patient_vitals_simulator.py` generates 20 patients and creates a new vitals reading every 
2 seconds for a random one of them — heart rate, oxygen level (SpO2), temperature, and 
blood pressure. The ranges are a bit wider than "healthy normal" on purpose, so some 
readings come out looking risky and the dashboard actually has variety to show.
About 10% of records get a deliberate error thrown in (a missing field, a negative heart 
rate, or an impossible SpO2 like 150) — this is just so the cleaning step later actually 
has something to clean. Each record gets turned into JSON and published to a Pub/Sub topic.

### 2. Bronze layer — raw data, untouched

Dataflow reads the messages straight off Pub/Sub, decodes them back into text, batches 
them into 60-second windows, and dumps them as-is into a `bronze/` folder in GCS. No 
cleaning here — this is just a raw backup of everything that came through.

### 3. Silver layer — cleaning it up

From that same stream, the pipeline:
- Turns the JSON back into a usable record
- Throws out anything broken (missing fields, negative heart rate, SpO2 over 100, etc.)
- Calculates a risk score using heart rate, temperature, and SpO2 (higher heart rate and 
  temperature = more risk; lower SpO2 = more risk, since low oxygen is bad)
- Labels each record Low / Moderate / High risk based on that score
- Saves the cleaned version to `silver/` in GCS

### 4. Gold layer — one summary per patient

Since each patient shows up in multiple records, this step groups everything by 
`patient_id` and averages out their heart rate, SpO2, and temperature. For risk level, it 
just takes the worst one they hit (if they were ever "High," they're marked High overall). 
This final summary gets appended into a BigQuery table.

### 5. Power BI dashboard

Power BI connects straight to the BigQuery table using DirectQuery. The dashboard has a dropdown to pick a patient, three gauges (heart rate, SpO2, 
temperature) that turn more red the riskier the number gets, and a card showing that patient's overall risk level.

