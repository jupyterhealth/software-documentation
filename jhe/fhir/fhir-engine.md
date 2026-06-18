# FHIR Engine

A small, config-driven engine that powers a FHIR R5 server over Django. The guiding rule:

> **A Django model backs only the JHE-system view of a FHIR resource. Everything else is
> stored opaquely in a single generic `FhirAuxResource` table.**

Concretely, the Django models hold:

| FHIR resource  | Django model (JHE system)                                                   | Everything else → `FhirAuxResource` |
| -------------- | --------------------------------------------------------------------------- | ----------------------------------- |
| `Observation`  | `Observation` — **OMH only** (`code` system `https://w3id.org/openmhealth`) | any other / code-less Observation   |
| `Device`       | `DataSource`                                                                | any other Device                    |
| `Group`        | `Study`                                                                     | any other Group                     |
| `Organization` | `Organization`                                                              | any other Organization              |
| `Patient`      | `Patient`                                                                   | any other Patient                   |
| `Practitioner` | `Practitioner`                                                              | any other Practitioner              |

Both kinds of resource are declared in [core/fhir/fhir_config.json](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/fhir_config.json).
A **mapped resource** is projected onto its Django model by a field mapping (read renders the
model through the mapping; the model is the system of record). An **auxiliary resource** has no
mapping — its whole FHIR body lives verbatim in `FhirAuxResource.fhir_data`.

Every incoming FHIR resource is validated against [`fhir.resources`](https://pypi.org/project/fhir.resources/)
(7.x) on the way in. Reads are not re-validated.

## Routing

Each HTTP request maps to a FHIR **interaction** (search / read / create / update / delete).
The config drives which backing store handles it, via two annotations:

- `__interaction` — the allow-list for a resource entry (a list of interactions, or `["*"]` for
  all). **Mapped** resources declare it as `meta.__interaction`; **auxiliary** entries declare it
  at the top level. Every entry (mapped and aux) must have one.
- `__criteria` — only on a mapped resource that exposes *all* interactions (so it would otherwise
  never fall back to aux). It is a predicate on the incoming resource that routes a **create**
  between the model and aux. Today only `Observation` uses it: `code=https://w3id.org/openmhealth|`.

Given a resource `R` with mapped interactions `M`, aux interactions `A`, and optional criteria `C`:

| Interaction                | Routing                                                                                                                                                                                      |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **search**                 | UNION of the mapped Django rows (if `search ∈ M`) and the `FhirAuxResource` rows of that type (if `search ∈ A`), in one searchset Bundle.                                                    |
| **read / update / delete** | by **id shape** — a **UUID** id targets `FhirAuxResource`; an **integer** id targets the mapped Django model. (FhirAuxResource uses a UUID primary key, so the two id spaces never collide.) |
| **create**                 | if `create ∈ M` and (`C` absent or `C` matches the payload) → mapped model; else if `create ∈ A` → aux; else `405`.                                                                          |

With the shipped config, `Device`/`Group`/`Organization`/`Patient`/`Practitioner` are
`read,search` against their model — so **all their writes fall through to `FhirAuxResource`** —
and `Observation` is `*` with the OMH criteria, so an **OMH** Observation create writes the
`Observation` model while **any other** Observation create lands in `FhirAuxResource`.

So searching `Group` returns the mapped `Study` rows **plus** any `Group` rows in
`FhirAuxResource`; the same holds for every mapped type.

## Searching mapped resources: the normalized `fhir_search`

Every mapped model exposes one uniform entry point that the generic handler calls for both
**search** and **read**:

```python
Model.fhir_search(jhe_user_id, resource_id=None, organization_id=None,
                  study_id=None, patient_id=None, **params) -> QuerySet[Model]
```

It returns a **lazy queryset of model instances** (the engine renders each through the config
mapping; formatting is never the model's job). The contract is identical across all six models:

1. **The user is resolved from `jhe_user_id`** via [`resolve_fhir_user`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/scope.py)
   (a single query, both role profiles `select_related`-ed). There is no `is_patient` flag —
   the method decides the branch itself, so handlers and tests just pass an id. An unknown id is
   a **404**.
1. **A patient user gets a self-scoped result.** The `organization_id` / `study_id` /
   `patient_id` filters are **ignored**; the method returns the rows that belong to that patient
   (their own observations, the studies/organizations/devices/practitioners they are attached to,
   or their own Patient record).
1. **A practitioner gets an organization-membership-scoped result.** The base queryset is anchored
   on the practitioner's organizations, then narrowed by whichever explicit filters are present.
   Each *targeted* filter is authorized up front by
   [`authorize_practitioner_scope`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/scope.py): an `organization_id` they do not belong
   to, a `study_id` under an organization they are not in, or a `patient_id` who shares no
   organization with them raises **403**. A **paramless** practitioner search returns *everything*
   across their authorized organizations (the "return-all" rule).
1. **`resource_id`** narrows to a single row by primary key. The view only ever passes it for a
   **read** of an **integer** id (UUID ids are routed to `FhirAuxResource` first), and it is
   applied *inside* the same authorization scope — so reading an id you may not see is a clean
   **404**, not a 403.
1. **`**params`** carries the non-location filters — `patient_identifier_system` /
   `patient_identifier_value` (Patient, Observation) and `coding_system` / `coding_code`
   (Observation). Every model accepts `**params` and ignores the keys it does not use. An
   **identifier is a search predicate, not a targeted resource**: it is *not* authorized (no 403)
   — the organization join already scopes the result, so an unmatched/unauthorized identifier just
   yields an empty set.
1. **Filters chain (AND).** Passing several at once narrows progressively.

### Query parameters → `fhir_search` kwargs

The generic handler ([`MappedResourceHandler._search_kwargs`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py)) translates the
canonical FHIR search params for every mapped resource:

| Query parameter                                                           | kwarg                                                    |
| ------------------------------------------------------------------------- | -------------------------------------------------------- |
| `?patient=<id>`                                                           | `patient_id`                                             |
| `?patient.organization=<id>`                                              | `organization_id`                                        |
| `?patient._has:Group:member:_id=<id>`                                     | `study_id`                                               |
| `?identifier=<system>\|<value>` / `?patient.identifier=<system>\|<value>` | `patient_identifier_system` / `patient_identifier_value` |
| `?code=<system>\|<value>`                                                 | `coding_system` / `coding_code`                          |
| path id `.../<resource>/<id>`                                             | `resource_id`                                            |

> **camelCase caveat:** the client sends the FHIR-standard `patient._has:Group:member:_id` (capital
> `Group`), but `djangorestframework_camel_case` snake-cases every incoming query-param key before
> it reaches `request.GET`, so the server actually reads **`patient._has:_group:member:_id`**. The
> other keys are already lowercase and pass through unchanged. See the note in `_search_kwargs`.

### Per-model behaviour

In every row below, the *practitioner* paths are additionally bounded by the practitioner's own
organization membership (and authorized, 403 on a targeted mismatch); the *patient* path ignores
the location filters and returns the self-scoped set.

| Model (resource)          | `organization_id`                    | `study_id`                                                                                            | `patient_id`                                      | extra `**params`                                                | patient user sees                           |
| ------------------------- | ------------------------------------ | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------- |
| **DataSource** (`Device`) | devices in studies under that org    | devices used in that study                                                                            | devices used in the studies that patient is in    | —                                                               | devices in the studies they are enrolled in |
| **Study** (`Group`)       | studies under that org               | the single study                                                                                      | studies that patient is enrolled in               | —                                                               | the studies they are enrolled in            |
| **Organization**          | the single org                       | the org backing that study                                                                            | the orgs that patient belongs to                  | —                                                               | the orgs they belong to                     |
| **Practitioner**          | practitioners in that org            | practitioners in that study's org                                                                     | practitioners in the orgs that patient belongs to | —                                                               | practitioners in the orgs they belong to    |
| **Patient**               | patients in that org                 | patients enrolled in that study                                                                       | the single patient                                | `identifier` → the patient with that identifier                 | only themselves                             |
| **Observation**           | observations of patients in that org | observations of patients enrolled in that study **whose code is one of the study's requested scopes** | that patient's observations                       | `identifier` → that patient's; `code` → matching `system\|code` | their own observations                      |

Notes:

- **Observation study scope** is the one place a `study_id` does more than membership: it requires
  the patient be enrolled in the study **and** the observation's `code` be one of that study's
  `StudyScopeRequest` codes (matched against the *same* study), so a multi-study patient only sees
  each study's consented codes.
- **Patient** no longer requires study enrollment — a practitioner sees *every* patient sharing one
  of their organizations (organization membership is the access boundary).
- **Implementation note (laziness):** `Patient` and `Organization` delegate their practitioner
  query to the existing lazy `for_*` helpers (reused by the `/api/v1` REST API); `Observation`,
  `Study`, `DataSource`, and `Practitioner` inline the practitioner query by `jhe_user_id` because
  their `for_*` helpers eagerly resolve the `Practitioner` via `get_object_or_404`, which would add
  a second query and break the single-query laziness the search relies on.

### `FhirAuxResource.fhir_search`

The auxiliary store follows the **same normalized contract**, with one extra required argument —
the `resource_type`, since the single `FhirAuxResource` table holds every aux type:

```python
FhirAuxResource.fhir_search(
    jhe_user_id,
    resource_type,
    resource_id=None,
    organization_id=None,
    study_id=None,
    patient_id=None,
    **params
)
```

Each aux row reaches its owning patient through its `FhirSource`
(`FhirAuxResource → FhirSource → Patient`), so the filters are expressed against
`fhir_source__patient`: `patient_id` → that patient's rows; `organization_id` / `study_id` → the
rows of all patients in that organization / study; `resource_id` → the single row by UUID. The
patient/practitioner split and the `authorize_practitioner_scope` 403s are identical to the mapped
models. The [`AuxResourceHandler`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py) calls it for both search and read, sharing
the same `_canonical_search_kwargs` query-param translation as the generic mapped handler — except
the `X-JHE-FHIR-Source-ID` header **wins** when present (it pins the read to that source's patient
and the query params are ignored). (Writes still resolve their target row through
`FhirAuxResource.for_patient`, since a write always names a source and therefore a concrete
patient.)

## Components

| File                                                                                                                                                                                                                                                               | Responsibility                                                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [core/fhir/fhir_config.json](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/fhir_config.json)                                                                                                                                         | Declares `mapped_resources` (field mappings + `meta.__interaction` / `__criteria`) and `aux_resources` (`resourceType` + `__interaction`).                                                                |
| [core/fhir/config.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/config.py)                                                                                                                                                       | Loads the JSON once at import; exposes `get_resource_mapping`, `mapped_interactions` / `aux_interactions`, `mapped_criteria`, `mapped_model_name`, and **`get_config_errors()`** (validation, see below). |
| [core/fhir/engine.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/engine.py)                                                                                                                                                       | The renderer: `build_fhir_resource` (model → FHIR dict), `render_resource`, `matches_criteria`, `expand_interactions`.                                                                                    |
| [core/fhir/fhir_validation.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/fhir_validation.py)                                                                                                                                     | `validate_fhir_resource(resource_type, data)` — parse an incoming FHIR body against its `fhir.resources` model (DRF 400 on failure).                                                                      |
| [core/serializers/observation.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/serializers/observation.py), [core/serializers/patient.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/serializers/patient.py) | `FHIRObservationSerializer` / `FHIRPatientSerializer` call the engine. (Observation Base64-encodes `valueAttachment.data` afterwards.)                                                                    |
| [core/serializers/aux_resource.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/serializers/aux_resource.py)                                                                                                                             | `FHIRAuxResourceSerializer` returns a `FhirAuxResource`'s stored body verbatim (with `resourceType`/`id` forced).                                                                                         |
| [core/fhir/scope.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/scope.py)                                                                                                                                                         | `resolve_fhir_user` (patient-vs-practitioner from the `jhe_user_id`) and `authorize_practitioner_scope` (403 on an unauthorized organization/study/patient), shared by every model's `fhir_search`.       |
| [core/views/fhir.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py)                                                                                                                                                         | `FHIRResourceView` — the unified endpoint, routing table, the generic mapped handler, and the aux handler.                                                                                                |
| [core/fhir/pagination.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/pagination.py)                                                                                                                                               | Wraps serialized resources in a FHIR `searchset` Bundle.                                                                                                                                                  |

## The configuration

`mapped_resources` and `aux_resources` are arrays of objects carrying a `"resourceType"`. A
mapped entry additionally holds its field mapping — a tree of dicts, lists, and strings, where
strings are tiny expressions (literal `"'final'"`, path `"DataSource.name"`, or `+`-concatenation
`"'Patient/' + Observation.subject_patient"`). The path prefix is the **Django model** backing the
resource (which can differ from the resourceType — a `Device` is a `DataSource`, a `Group` a
`Study`). Output keys are FHIR field names in camelCase. (The rendering rules — fan-out of related
managers via `as_fhir_element()`, materializing a single FK to its pk, and pruning empty
leaves/templates — are unchanged from the original engine; see the code comments in
[core/fhir/engine.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/engine.py).)

### Validation (`get_config_errors`, lazy, 500 on failure)

[`FHIRResourceView`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py) calls `get_config_errors()` on each request (cached) and
returns a **500 OperationOutcome** listing any problems. The five checks
([core/fhir/config.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/config.py)):

1. **Every** entry — mapped and aux — has a non-empty `__interaction`.
1. Each interaction is one of `create/read/update/delete/search` or `"*"`.
1. A mapped resource whose interactions cover everything (`"*"`) **must** declare `__criteria`
   (otherwise it could never fall back to aux).
1. **Every path resolves** on the backing model: the model is the path prefix (resolved via
   `apps.get_model("core", name)`), and each dotted segment must be a field, a `@property`, or an
   FK hop (e.g. `Patient.jhe_user.email`, `Observation.codeable_concepts`).
1. **Every field name is valid FHIR**: each non-`__` key of a mapped resource must be a real
   element of the matching `fhir.resources` model (`ModelClass.elements_sequence()`).

## Auxiliary resources, FhirSource & the source header

`FhirAuxResource` ([core/models/fhir_aux_resource.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/models/fhir_aux_resource.py)) stores the
whole FHIR body in `fhir_data`, served with full CRUD and no computation. Key points:

- The **primary key is a UUID**, so the FHIR `id` is a UUID — disjoint from the integer pks of the
  mapped models, which is what makes id-shape routing unambiguous.

- **Every row links to a `FhirSource` (required)** and, through it, to a `patient`.

- Writes are validated against `fhir.resources`; the incoming body (snake-cased by the camel-case
  parser) is re-camelized before validation/storage so `fhir_data` is valid FHIR.

- On every write, two best-effort columns are populated from the body (both may be null):

  - `fhir_resource_id` ← the resource's own `id`;
  - `patient_fhir_id` ← the referenced Patient id: the resource `id` itself when
    `resourceType == Patient`, else the `Patient/<id>` in `subject.reference`, `patient.reference`,
    or `beneficiary.reference` (first match wins).

- On every write, the stored body is also **stamped with JHE provenance extensions**
  (`apply_jhe_extensions` in [core/models/fhir_aux_resource.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/models/fhir_aux_resource.py),
  applied by `_persist_aux` in [core/views/fhir.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py))
  so a reader can attribute an opaque aux body to its source and patient without a join. Three
  `extension` entries (base URL `https://jupyterhealth.org/fhir/StructureDefinition/`) are added:

  - `.../fhir-source-id` — `valueInteger`, the `FhirSource` pk;
  - `.../patient-id` — `valueInteger`, the owning patient's pk (from the source);
  - `.../patient-full-name` — `valueString`, the patient's `name_given name_family` (omitted
    entirely when the patient has no name).

  Any prior copies of these three URLs are stripped first, so re-stamping on update (or a re-seed)
  replaces the values rather than accumulating duplicates; other extensions on the body are left
  untouched. The columns above are computed *before* `_aux_body` strips `resourceType`; the
  extensions are added *after* `fhir.resources` validation, so they never affect the incoming-body
  check. The same helper is used by the seed command so freshly seeded aux rows carry the
  extensions too.

- On every write, the stored body is also **stamped with JHE provenance extensions**
  (`apply_jhe_extensions` in [core/models/fhir_aux_resource.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/models/fhir_aux_resource.py),
  applied by `_persist_aux` in [core/views/fhir.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py))
  so a reader can attribute an opaque aux body to its source and patient without a join. Three
  `extension` entries (base URL `https://jupyterhealth.org/fhir/StructureDefinition/`) are added:

  - `.../fhir-source-id` — `valueInteger`, the `FhirSource` pk;
  - `.../patient-id` — `valueInteger`, the owning patient's pk (from the source);
  - `.../patient-full-name` — `valueString`, the patient's `name_given name_family` (omitted
    entirely when the patient has no name).

  Any prior copies of these three URLs are stripped first, so re-stamping on update (or a re-seed)
  replaces the values rather than accumulating duplicates; other extensions on the body are left
  untouched. The columns above are computed *before* `_aux_body` strips `resourceType`; the
  extensions are added *after* `fhir.resources` validation, so they never affect the incoming-body
  check. The same helper is used by the seed command so freshly seeded aux rows carry the
  extensions too.

### FhirSource

A `FhirSource` ([core/models/fhir_source.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/models/fhir_source.py)) is an upstream FHIR source
a **patient registers for themselves** (fields: `patient`, `data_source`, `label`, `fhir_base_url`)
before uploading FHIR resources. CRUD lives at `api/v1/fhir_sources` via
[`FhirSourceViewSet`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir_source.py), scoped to the requesting patient (their `patient`
is assigned server-side).

### The `X-JHE-FHIR-Source-ID` header

The `X-JHE-FHIR-Source-ID` header names the `FhirSource`, from which a request resolves
`(patient, fhir_source)` ([resolve_fhir_source_context](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py)):

- **patient users** are scoped to themselves via the access token; the named source must be theirs;
- **non-patient users** (practitioners) take the patient from the source's `patient`, authorized by
  organization sharing.

An unknown source is **400** and a source the user may not use is **403**. How the header is treated
depends on the interaction:

- **Writes (create/update/delete) require the header** — a missing one is **400**. The new/edited row
  is linked to the named source and its patient.
- **Reads (search/read) treat it as optional** — they go through the normalized
  `FhirAuxResource.fhir_search` (see below), which does the same patient-vs-practitioner split and
  organization-membership scoping as the mapped models. **The header wins:** when present it scopes
  the read to that source's patient (resolving the source also authorizes the caller) and the
  canonical query params are ignored. When absent, the canonical `patient` / `patient.organization`
  / `patient._has:Group:member:_id` filters apply — so e.g.
  `GET /FHIR/R5/QuestionnaireResponse?patient._has:Group:member:_id=<study>` returns that resource
  type's aux rows for the study's patients the practitioner can access — and otherwise `fhir_search`
  returns every aux resource the user can access (a practitioner's organization patients, or a
  patient user's own). The aux portion of a **union search** on a mapped+aux type is therefore
  always included.

## The unified endpoint

A single view, [`FHIRResourceView`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py), serves every supported resource at
`FHIR/<version>/<resource>` and `.../<resource>/<id>` (`<version>` is the config `fhir_version`,
e.g. `FHIR/R5/Patient`); the lowercase `fhir/r5/` path is a backward-compatible alias. It applies
the routing table above, dispatching to the generic **mapped handler** (which translates the
canonical search params into the model's `fhir_search` and renders each row through the config
mapping; `ObservationHandler` subclasses it only for the Base64 serializer and OMH create) or the
**aux handler**. The FHIR bundle batch stays at `POST` on the
base (`FHIR/R5/`), served by [`FHIRBase`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir_base.py), which routes each Observation
entry by the same OMH criteria. Domain and DRF exceptions are rendered as a FHIR `OperationOutcome`
with the right status by `handle_exception`.

## Adding a resource

- **A new aux-only resource**: add `{ "resourceType": "<Name>", "__interaction": ["*"] }` to
  `aux_resources` and restart. No model/serializer/handler changes.
- **A new mapped resource**: add an entry to `mapped_resources` (field mapping + `meta.__interaction`,
  and `__criteria` if it is fully writable), add a matching aux entry if its non-system rows should
  fall through, and give the backing model a `fhir_search(jhe_user_id, resource_id=None, organization_id=None, study_id=None, patient_id=None, **params)` returning instances (use
  [`resolve_fhir_user`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/scope.py) + [`authorize_practitioner_scope`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/scope.py)
  for the patient/practitioner split and the 403s). The generic handler then serves it with no view
  changes; only register a subclass in `_MAPPED_HANDLERS` if it needs custom serialize/create (as
  Observation does). Add model hooks (`as_fhir_element()`, an iterable property) where the FHIR shape
  differs from the columns.

## Tests

- Engine/serializer shape: `FHIRPatientSerializerTests`, `FHIRObservationSerializerTests`
  ([tests/backend/test_model_methods.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/tests/backend/test_model_methods.py)).
- Config validation: [tests/backend/test_fhir_config.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/tests/backend/test_fhir_config.py).
- Routing, union search, aux CRUD, source-header resolution, id-shape routing:
  [tests/backend/test_fhir_resource_view.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/tests/backend/test_fhir_resource_view.py).
- End-to-end pagination + Bundle validation: `test_patient_pagination` /
  `test_observation_pagination`, validating each page against `fhir.resources.bundle.Bundle`.
