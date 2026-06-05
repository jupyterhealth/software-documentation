# FHIR Engine

A small, config-driven engine that powers a FHIR R5 server over Django. The guiding rule:

> **A Django model backs only the JHE-system view of a FHIR resource. Everything else is
> stored opaquely in a single generic `FhirAuxResource` table.**

Concretely, the Django models hold:

| FHIR resource | Django model (JHE system) | Everything else → `FhirAuxResource` |
| --- | --- | --- |
| `Observation` | `Observation` — **OMH only** (`code` system `https://w3id.org/openmhealth`) | any other / code-less Observation |
| `Device` | `DataSource` | any other Device |
| `Group` | `Study` | any other Group |
| `Organization` | `Organization` | any other Organization |
| `Patient` | `Patient` | any other Patient |
| `Practitioner` | `Practitioner` | any other Practitioner |

Both kinds of resource are declared in [core/fhir/fhir_config.json](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/fhir/fhir_config.json).
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

| Interaction | Routing |
| --- | --- |
| **search** | UNION of the mapped Django rows (if `search ∈ M`) and the `FhirAuxResource` rows of that type (if `search ∈ A`), in one searchset Bundle. |
| **read / update / delete** | by **id shape** — a **UUID** id targets `FhirAuxResource`; an **integer** id targets the mapped Django model. (FhirAuxResource uses a UUID primary key, so the two id spaces never collide.) |
| **create** | if `create ∈ M` and (`C` absent or `C` matches the payload) → mapped model; else if `create ∈ A` → aux; else `405`. |

With the shipped config, `Device`/`Group`/`Organization`/`Patient`/`Practitioner` are
`read,search` against their model — so **all their writes fall through to `FhirAuxResource`** —
and `Observation` is `*` with the OMH criteria, so an **OMH** Observation create writes the
`Observation` model while **any other** Observation create lands in `FhirAuxResource`.

So searching `Group` returns the mapped `Study` rows **plus** any `Group` rows in
`FhirAuxResource`; the same holds for every mapped type.

## Components

| File | Responsibility |
| --- | --- |
| [core/fhir/fhir_config.json](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/fhir/fhir_config.json) | Declares `mapped_resources` (field mappings + `meta.__interaction` / `__criteria`) and `aux_resources` (`resourceType` + `__interaction`). |
| [core/fhir/config.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/fhir/config.py) | Loads the JSON once at import; exposes `get_resource_mapping`, `mapped_interactions` / `aux_interactions`, `mapped_criteria`, `mapped_model_name`, and **`get_config_errors()`** (validation, see below). |
| [core/fhir/engine.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/fhir/engine.py) | The renderer: `build_fhir_resource` (model → FHIR dict), `render_resource`, `matches_criteria`, `expand_interactions`. |
| [core/fhir/fhir_validation.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/fhir/fhir_validation.py) | `validate_fhir_resource(resource_type, data)` — parse an incoming FHIR body against its `fhir.resources` model (DRF 400 on failure). |
| [core/serializers/observation.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/serializers/observation.py), [core/serializers/patient.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/serializers/patient.py) | `FHIRObservationSerializer` / `FHIRPatientSerializer` call the engine. (Observation Base64-encodes `valueAttachment.data` afterwards.) |
| [core/serializers/aux_resource.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/serializers/aux_resource.py) | `FHIRAuxResourceSerializer` returns a `FhirAuxResource`'s stored body verbatim (with `resourceType`/`id` forced). |
| [core/views/fhir.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/views/fhir.py) | `FHIRResourceView` — the unified endpoint, routing table, mapped handlers, and the aux handler. |
| [core/fhir/pagination.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/fhir/pagination.py) | Wraps serialized resources in a FHIR `searchset` Bundle. |

## The configuration

`mapped_resources` and `aux_resources` are arrays of objects carrying a `"resourceType"`. A
mapped entry additionally holds its field mapping — a tree of dicts, lists, and strings, where
strings are tiny expressions (literal `"'final'"`, path `"DataSource.name"`, or `+`-concatenation
`"'Patient/' + Observation.subject_patient"`). The path prefix is the **Django model** backing the
resource (which can differ from the resourceType — a `Device` is a `DataSource`, a `Group` a
`Study`). Output keys are FHIR field names in camelCase. (The rendering rules — fan-out of related
managers via `as_fhir_element()`, materializing a single FK to its pk, and pruning empty
leaves/templates — are unchanged from the original engine; see the code comments in
[core/fhir/engine.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/fhir/engine.py).)

### Validation (`get_config_errors`, lazy, 500 on failure)

[`FHIRResourceView`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/views/fhir.py) calls `get_config_errors()` on each request (cached) and
returns a **500 OperationOutcome** listing any problems. The five checks
([core/fhir/config.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/fhir/config.py)):

1. **Every** entry — mapped and aux — has a non-empty `__interaction`.
2. Each interaction is one of `create/read/update/delete/search` or `"*"`.
3. A mapped resource whose interactions cover everything (`"*"`) **must** declare `__criteria`
   (otherwise it could never fall back to aux).
4. **Every path resolves** on the backing model: the model is the path prefix (resolved via
   `apps.get_model("core", name)`), and each dotted segment must be a field, a `@property`, or an
   FK hop (e.g. `Patient.jhe_user.email`, `Observation.codeable_concepts`).
5. **Every field name is valid FHIR**: each non-`__` key of a mapped resource must be a real
   element of the matching `fhir.resources` model (`ModelClass.elements_sequence()`).

## Auxiliary resources, FhirSource & the source header

`FhirAuxResource` ([core/models/fhir_aux_resource.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/models/fhir_aux_resource.py)) stores the
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

### FhirSource

A `FhirSource` ([core/models/fhir_source.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/models/fhir_source.py)) is an upstream FHIR source
a **patient registers for themselves** (fields: `patient`, `data_source`, `label`, `fhir_base_url`)
before uploading FHIR resources. CRUD lives at `api/v1/fhir_sources` via
[`FhirSourceViewSet`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/views/fhir_source.py), scoped to the requesting patient (their `patient`
is assigned server-side).

### The `X-JHE-FHIR-Source-ID` header

The `X-JHE-FHIR-Source-ID` header names the `FhirSource`, from which a request resolves
`(patient, fhir_source)` ([resolve_fhir_source_context](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/views/fhir.py)):

- **patient users** are scoped to themselves via the access token; the named source must be theirs;
- **non-patient users** (practitioners) take the patient from the source's `patient`, authorized by
  organization sharing.

An unknown source is **400** and a source the user may not use is **403**. How the header is treated
depends on the interaction:

- **Writes (create/update/delete) require the header** — a missing one is **400**. The new/edited row
  is linked to the named source and its patient.
- **Reads (search/read) treat it as optional** — when present, the response is scoped to that source's
  patient; when absent, it returns every aux resource the user can access (a practitioner's
  organization patients via `FhirAuxResource.fhir_search`, or a patient user's own). The aux portion
  of a **union search** on a mapped+aux type is therefore always included.

## The unified endpoint

A single view, [`FHIRResourceView`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/views/fhir.py), serves every supported resource at
`FHIR/<version>/<resource>` and `.../<resource>/<id>` (`<version>` is the config `fhir_version`,
e.g. `FHIR/R5/Patient`); the lowercase `fhir/r5/` path is a backward-compatible alias. It applies
the routing table above, dispatching to a **mapped handler** (per-resource scoped queryset + read,
plus Observation's OMH create) or the **aux handler**. The FHIR bundle batch stays at `POST` on the
base (`FHIR/R5/`), served by [`FHIRBase`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/views/fhir_base.py), which routes each Observation
entry by the same OMH criteria. Domain and DRF exceptions are rendered as a FHIR `OperationOutcome`
with the right status by `handle_exception`.

## Adding a resource

- **A new aux-only resource**: add `{ "resourceType": "<Name>", "__interaction": ["*"] }` to
  `aux_resources` and restart. No model/serializer/handler changes.
- **A new mapped resource**: add an entry to `mapped_resources` (field mapping + `meta.__interaction`,
  and `__criteria` if it is fully writable), add a matching aux entry if its non-system rows should
  fall through, give the backing model a `fhir_search` returning instances, register a handler in
  `_MAPPED_HANDLERS`, and add model hooks (`as_fhir_element()`, an iterable property) where the FHIR
  shape differs from the columns.

## Tests

- Engine/serializer shape: `FHIRPatientSerializerTests`, `FHIRObservationSerializerTests`
  ([tests/backend/test_model_methods.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/tests/backend/test_model_methods.py)).
- Config validation: [tests/backend/test_fhir_config.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/tests/backend/test_fhir_config.py).
- Routing, union search, aux CRUD, source-header resolution, id-shape routing:
  [tests/backend/test_fhir_resource_view.py](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/tests/backend/test_fhir_resource_view.py).
- End-to-end pagination + Bundle validation: `test_patient_pagination` /
  `test_observation_pagination`, validating each page against `fhir.resources.bundle.Bundle`.
