---
title: Demo Data
---

### Demo Data

JupyterHealth Exchange (JHE) ships two commands that generate synthetic time-series data: continuous glucose monitor (CGM) readings plus Oura-style wearable records. Both use the same generator, so the data looks the same either way. They differ only in who receives it.

| Command                                                | Who gets data                  | When to use it                                                  |
| ------------------------------------------------------ | ------------------------------ | --------------------------------------------------------------- |
| `python manage.py seed --flush-db --with-rich-demo`    | Creates 20 new demo patients   | Standing up a fresh environment with a full cohort              |
| `python manage.py seed_patient_demo --patient-id <id>` | One patient who already exists | Giving an existing patient history, without wiping the database |

All generated data is clearly synthetic. Every record carries an `acquisition_provenance.source_name` of `demo-cgm-generator` or `demo-synthea-omh-ieee-generator`, so it can always be told apart from real data.

#### What gets generated

Per patient: 14 days of CGM readings every 15 minutes (1,345 records), plus 60 to 180 days of daily wearable records across 8 types. Everything ends on the day the command is run.

| Scope                          | Source |
| ------------------------------ | ------ |
| `omh:blood-glucose:4.0`        | Dexcom |
| `omh:physical-activity:1.2`    | Oura   |
| `omh:step-count:3.0`           | Oura   |
| `ieee:sleep-stage-summary:1.0` | Oura   |
| `omh:sleep-episode:1.1`        | Oura   |
| `omh:sleep-duration:2.0`       | Oura   |
| `omh:heart-rate:2.0`           | Oura   |
| `omh:respiratory-rate:2.0`     | Oura   |
| `omh:oxygen-saturation:2.0`    | Oura   |

Glucose and sleep values are driven by an age-based risk score, so older patients get higher baselines, larger meal spikes and worse sleep. The values are plausible, not clinical-grade.

#### Seeding one existing patient

`--patient-id` is the **Patient ID**, not the User ID. See [Patient Identifiers](./patient-identifiers.md) if you are unsure which you have.

The command creates or reuses a study named "CGM & Wearables Demo" in an organization the patient already belongs to, enrolls the patient, records consent for all 9 scopes, then generates the observations.

It refuses to run, and changes nothing, in these cases:

| Message                                       | Meaning                                                                                     |
| --------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `--patient-id is required.`                   | No id was given                                                                             |
| `No Patient with id <n>.`                     | The id does not match a patient. You may have used the User ID                              |
| `Patient <n> has no usable birth date`        | The birth date is missing or under a year old, so the age-based trends would be meaningless |
| `Patient <n> belongs to no Organization.`     | Nobody could see the data, since access flows through the organization                      |
| `Missing CodeableConcept ... run seed first.` | The base seed has not run, so the scopes do not exist yet                                   |
| `Patient <n> already has demo data.`          | Data is already there. Remove it before generating a new window                             |

#### Who can see the generated data

Organization membership is flat. It does **not** follow the organization tree. A practitioner sees the data only if they have a role in the exact organization the patient belongs to, which the success line names:

```
Patient demo seeded: patient 40001, 2305 observations in study 'CGM & Wearables Demo' under organization 'Lifespan Lab' (id 20003).
```

If the patient belongs to several organizations, the lowest-numbered one is used. The count varies per patient, because the wearable history length is drawn from the 60 to 180 day range.

#### Verifying it worked

| Step                     | Command                                                    | Expected                                               |
| ------------------------ | ---------------------------------------------------------- | ------------------------------------------------------ |
| 1. Generate              | `python manage.py seed_patient_demo --patient-id 40001`    | The success line above                                 |
| 2. Data is there         | `GET /api/v1/observations?patient_id=40001`                | Records returned, count in the thousands               |
| 3. Study wiring is right | `GET /api/v1/observations?patient_id=40001&study_id=<id>`  | The same data, not an empty list                       |
| 4. Time filters work     | `GET /FHIR/R5/Observation?patient=40001&date=ge2026-01-01` | A non-empty Bundle                                     |
| 5. Charts have shape     | Open the patient in the admin UI, under OMH Observations   | Glucose rises after meals, sleep varies night to night |

Step 3 catches a half-wired study and step 4 catches missing timestamps. Both look fine in step 2 even when they are broken.

The `/api/v1/observations` filters are snake_case (`patient_id`, `study_id`). A camelCase key is ignored rather than rejected, so `?patientId=...` quietly returns every patient you can see. Results are paginated at 20 per page by default, so a short list is pagination rather than missing data.

Data generated by `seed --with-rich-demo` does not populate the queryable timing columns, so FHIR searches using `?date=` return nothing for it. Data from `seed_patient_demo` does.
