---
title: Architecture
---

## Django

Django is a mature and well-supported web framework but was specifically chosen due to resourcing requirements. There are a few accommodations that had to be made for Django to support FHIR as described below.

### camelCase

- FHIR uses camelCase whereas Django uses snake_case.
- The [djangorestframework-camel-case](https://github.com/vbabiy/djangorestframework-camel-case) library is used to support camelCase but the conversion happens downstream whereas the schema validation happens upstream, so manually calling `humps` is also required in parts.

### DRF Serializers and Pydantic

- The Django Rest Framework uses the concept of Serializers to validate schemas, whereas the FHIR validator uses Pydantic.
- It is not reasonable to re-write the entire validation in the Serializer, so instead a combination of the two are used:
  - Top-level fields (most importantly the `id` of a record) are managed by the Serializer.
  - Nested fields (for example `code{}.coding[].system` above) are configured as a JSON field in the Serializer (so the top level field is this example is `code`) and then Pydantic is used to validate the whole schema including nested JSON.

- The [drf-pydantic](https://github.com/georgebv/drf-pydantic) library may allow Pydantic to be used as a Serializer, but this needs to be explored further.

### JSON Responses

- Postgres has rich JSON support allowing responses to be built directly from a raw Django SQL queries rather than using another layer of transforming logic.

## Single Page App (SPA) Web UI

A hard requirement was to avoid additional servers and frameworks (eg npm, react, etc) for the front end Web UI. Django supports traditional server-side templating but a modern Single Page App is better suited to this use case of interacting with the Admin REST API. For these reasons, a simple Vanilla JS SPA has been developed using [handlebars](https://github.com/handlebars-lang/handlebars.js) to render client side views from static HTML served using Django templates. The only other additional dependencies are [oidc-clinet-ts](https://github.com/authts/oidc-client-ts) for auth and [bootstrap](https://github.com/twbs/bootstrap) for styling.

## Data Model

```{caution} To be updated
```

```mermaid
erDiagram
    "users (FHIR Person)" ||--|{ "user_organizations": ""
    "users (FHIR Person)" ||--|{ "study_practitioners": ""
    "users (FHIR Person)" {
        int id
        jsonb identifer
        varchar password
        varchar name_family
        varchar name_given
        varchar telecom_email
    }
    "organizations (FHIR Organization)" ||--|{ "organizations (FHIR Organization)": ""
    "organizations (FHIR Organization)" ||--|{ "user_organizations": ""
    "organizations (FHIR Organization)" ||--|{ "smart_client_configs": ""
    "organizations (FHIR Organization)" ||--|{ "studies (FHIR Group)": ""
    "organizations (FHIR Organization)" {
        int id
        jsonb identifer
        varchar name
        enum type
        int part_of
    }
    "smart_client_configs" {
        int id
        int organization_id
        varchar well_known_uri
        varchar client_id
        varchar scopes
    }
    "user_organizations" {
        int id
        int user_id
        int organization_id
    }
    "patients (FHIR Patient)" ||--|| "users (FHIR Person)": ""
    "patients (FHIR Patient)" ||--|{ "observations (FHIR Observation)": ""
    "patients (FHIR Patient)" ||--|{ "study_patients": ""
    "patients (FHIR Patient)" {
        int id
        int user_id
        int organization_id
        varchar identifer
        varchar name_family
        varchar name_given
        date   birth_date
        varchar telecom_cell
    }
    "studies (FHIR Group)" ||--|{ "study_patients": ""
    "studies (FHIR Group)" ||--|{ "study_practitioners": ""
    "studies (FHIR Group)" ||--|{ "study_scope_requests": ""
    "studies (FHIR Group)" ||--|{ "study_data_sources": ""
    "studies (FHIR Group)" {
        int id
        int organization_id
        jsonb identifer
        varchar name
        varchar description
    }
    "study_patients" ||--|{ "study_patient_scope_consents": ""
    "study_patients" {
        int id
        int study_id
        int pateint_id
    }
    "study_scope_requests" ||--|{ "codeable_concepts (FHIR CodeableConcept)": ""
    "study_scope_requests" {
        int id
        int study_id
        enum scope_action
        int scope_code_id
    }

    "study_practitioners" {
        int id
        int study_id
        int user_id
    }
    
    "observations (FHIR Observation)" ||--|| "codeable_concepts (FHIR CodeableConcept)": ""
    "observations (FHIR Observation)" ||--|{ "observation_identifiers": ""
    "observations (FHIR Observation)" ||--|| "data_sources": ""
    "observations (FHIR Observation)" {
        int id
        int subject_patient_id
        int codeable_concept_id
        jsonb value_attachment_data
        timestamp transaction_time
    }

    "observation_identifiers" {
        int id
        int observation_id
        varchar system
        varchar value
    }
    
    "studies (FHIR Group)" ||--|{ "study_patients": ""
    "codeable_concepts (FHIR CodeableConcept)" {
        int id
        varchar coding_system
        varchar coding_code
        varchar text
    }
    "study_patient_scope_consents" ||--|| "codeable_concepts (FHIR CodeableConcept)": ""
    "study_patient_scope_consents" {
        int id
        int study_patient_id
        enum scope_action
        int scope_code_id
        bool consented
        timestamp consented_time       
    }
    "data_sources" ||--|{ "data_source_supported_scopes": ""
    "data_sources" ||--|{ "study_data_sources": ""
    "data_sources" {
        int id
        varchar name
        enum type
    }
    "data_source_supported_scopes" ||--|| "codeable_concepts (FHIR CodeableConcept)": ""
    "data_source_supported_scopes" {
        int id
        int data_source_id
        int scope_code_id
    }
    "study_data_sources" {
        int id
        int study_id
        int data_source_id
    }
```
