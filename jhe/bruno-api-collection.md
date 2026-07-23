---
title: Bruno API Collection
---

JHE ships a [Bruno](https://www.usebruno.com/) API collection for exercising the Admin and FHIR APIs by hand. Every request uses environment variables, so there are no hardcoded IDs.

| What                                | Where (in the JHE repo)                        |
| ----------------------------------- | ---------------------------------------------- |
| Collection (Admin + FHIR requests)  | `opencollection/JHE/`                          |
| Token-minting command               | `core/management/commands/create_bruno_app.py` |
| Test that keeps the collection true | `tests/backend/test_bruno_collection.py`       |

## Setup

1. Install [Bruno](https://www.usebruno.com/downloads).

1. Seed the database and mint an access token:

   ```bash
   python manage.py seed
   python manage.py create_bruno_app --email admin@example.com
   ```

   `create_bruno_app` creates a public OAuth app, generates a 1-year access token, and makes the user a manager of every organization. Copy the printed token.

1. In Bruno, choose **Open Collection** and select `opencollection/JHE/`.

1. Create an environment and set the variables below. Get real IDs from the Web UI or your seeded data.

   | Variable         | Meaning                                   |
   | ---------------- | ----------------------------------------- |
   | `BASE_URL`       | Server root, e.g. `http://localhost:8000` |
   | `ACCESS_TOKEN`   | Token printed by `create_bruno_app`       |
   | `ORG_ID`         | An organization id                        |
   | `PATIENT_ID`     | A patient id                              |
   | `STUDY_ID`       | A study id                                |
   | `USER_ID`        | A user id                                 |
   | `DATA_SOURCE_ID` | A data source id                          |
   | `CLIENT_APP_ID`  | An OAuth client (JheClient) id            |

1. Select the environment, then send any request.

The token expires after one year. Re-run `create_bruno_app` to mint a new one.

## Collection structure

| Folder    | Covers                                                                                                 |
| --------- | ------------------------------------------------------------------------------------------------------ |
| **Admin** | Organizations, Patients, Studies, Users, Practitioners, Data Sources, Clients, Settings, Observations  |
| **FHIR**  | FHIR R5 Patient, Observation (filtered search + `_summary=count`), and QuestionnaireResponse endpoints |

## Keeping it in sync

`test_bruno_collection.py` parses every request and fires it against the API, so any request that drifts from the real endpoints fails the backend test suite. Run it with the normal backend tests.
