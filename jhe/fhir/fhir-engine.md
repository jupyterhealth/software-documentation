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

| Interaction                | Routing                                                                                                                                                                                       |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **search**                 | exactly **one** store, chosen by the [`_source`](#the-metasource-discriminator--the-_source-search-param) param — never a union. `_source` absent or the JHE-native URI → the mapped Django rows; a `.../fhir-source/<id>` URI → that source's `FhirAuxResource` rows; `_source:below=.../fhir-source/` → every imported aux row. |
| **read / update / delete** | by **id shape** — a **UUID** id targets `FhirAuxResource`; an **integer** id targets the mapped Django model. (FhirAuxResource uses a UUID primary key, so the two id spaces never collide.) |
| **create**                 | if `create ∈ M` and (`C` absent or `C` matches the payload) → mapped model; else if `create ∈ A` → aux; else `405`.                                                                          |

With the shipped config, `Device`/`Group`/`Organization`/`Patient`/`Practitioner` are
`read,search` against their model — so **all their writes fall through to `FhirAuxResource`** —
and `Observation` is `*` with the OMH criteria, so an **OMH** Observation create writes the
`Observation` model while **any other** Observation create lands in `FhirAuxResource`.

**Search never spans both stores.** A single query resolves to one table so that filtering and
sorting push down to the database on that one table rather than merging two result sets in memory.
The client chooses which store with `_source`: a bare `GET /Group` returns the mapped `Study` rows
(JHE-native is the default); the imported `Group` rows in `FhirAuxResource` are reached with
`?_source=https://jupyterhealth.org/fhir/fhir-source/<id>` (one source) or
`?_source:below=https://jupyterhealth.org/fhir/fhir-source/` (all imported). The same holds for
every mapped type. (`read`/`update`/`delete` are single-table via id shape.)

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
    fhir_source_id=None,
    **params
)
```

Each aux row reaches its owning patient through its `FhirSource`
(`FhirAuxResource → FhirSource → Patient`), so the filters are expressed against
`fhir_source__patient`: `patient_id` → that patient's rows; `organization_id` / `study_id` → the
rows of all patients in that organization / study; `resource_id` → the single row by UUID;
`fhir_source_id` → the single upstream source (the `_source=.../fhir-source/<id>` read route).
Like an identifier, `fhir_source_id` is an **unauthorized predicate** — the organization/patient
join already scopes the result, so an inaccessible source simply yields nothing. The
patient/practitioner split and the `authorize_practitioner_scope` 403s are identical to the mapped
models. The [`AuxResourceHandler`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py) calls it for both search and read, sharing
the same `_canonical_search_kwargs` query-param translation as the generic mapped handler. The
`X-JHE-FHIR-Source-ID` header is **write-only** — it is ignored on reads (which resolve their store
and single-source filter from `_source`; see below). (Writes still resolve their target row
through `FhirAuxResource.for_patient`, since a write always names a source and therefore a concrete
patient.)

## Search parameters, `_sort` & `_summary`

On top of the store selection above, the endpoint applies the
[US Core "supported searches"](https://hl7.org/fhir/us/core/CapabilityStatement-us-core-server.html)
for each resource, plus `_sort` and `_summary`. Because a search has already resolved to **one**
store, every filter, sort, and count runs against that single table — mapped rows via the Django
ORM, auxiliary rows via a small Postgres JSONB query builder. Nothing merges in memory.

Three tiers of parameters, applied in this order to the authorized queryset the store returned:

1. **Resource-agnostic** — `_id` and `_lastUpdated` (`apply_common_search_filters` in
   [core/views/fhir.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py)). Both stores expose an `id` and a `last_updated`
   column, so these are plain ORM filters. `_id` is an exact match whose value shape decides the
   store anyway (a UUID never matches a mapped row, an integer never matches an aux row);
   `_lastUpdated` takes the `ge`/`le`/`gt`/`lt` comparators or a bare date (that whole day), and
   repeats AND together into a range.
1. **Location filters** — `patient` / `patient.organization` / `patient._has:Group:member:_id`,
   already translated to `fhir_search` kwargs (see the tables above). These are the
   authorization-scoping filters and are **not** re-expressed as body predicates.
1. **Resource-specific US Core params** + `_sort` — declared per resource as a `__search` block in
   the config and applied by [core/fhir/search.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/search.py). This is the rest of this section.

### The `__search` config block

Each resource entry (mapped or aux) may carry a `__search` object mapping a US Core search-param
name to a `{ "type": …, "path": … }` spec. The **path means different things per store** — a
mapped path is a Django field/lookup on the backing model; an aux path is a dotted FHIRPath into the
stored `fhir_data` body (camelCase) — because a resource is defined once per store, so each entry
carries the form its store needs. Example (`Condition`, aux):

```json
"__search": {
  "clinical-status": { "type": "token",     "path": "clinicalStatus.coding" },
  "category":        { "type": "token",     "path": "category.coding" },
  "code":            { "type": "token",     "path": "code.coding" },
  "encounter":       { "type": "reference", "path": "encounter.reference" },
  "onset-date":      { "type": "date",      "path": ["onsetDateTime", "onsetPeriod.start"] },
  "recorded-date":   { "type": "date",      "path": ["recordedDate"] }
},
"__sortDate": ["recordedDate"]
```

A param that is not declared for a resource is simply **ignored** (FHIR permits a server to ignore
unsupported search parameters), never an error. Which params each resource declares is taken
straight from the US Core CapabilityStatement's supported-searches set.

**Search-param types** (validated by `get_config_errors`):

| type         | Matches                                                                 | Store semantics |
| ------------ | ---------------------------------------------------------------------- | --------------- |
| `token`      | a `system\|code` against a `Coding`/`CodeableConcept.coding` (path → the coding array/element) | aux: `@.code`(& `@.system`) equality; mapped: exact match on the code part |
| `identifier` | like `token` but against an `Identifier` (`@.value` instead of `@.code`) | aux only |
| `code`       | a plain FHIR `code` scalar (`status`, `intent`, …); the token's system is ignored | aux: `@ == code` |
| `string`     | case-insensitive **starts-with** over one or more paths                 | aux: `like_regex`; mapped: `__istartswith` |
| `reference`  | a full `Type/id` **or** a bare id (any `…/id`)                          | aux only |
| `date`       | `ge`/`le`/`gt`/`lt` comparator or a bare date (prefix); polymorphic `[x]` paths are `COALESCE`d | both |
| `const`      | the mapped resource renders this element as a fixed literal (e.g. Observation `status` = `final`, Device `type` = `data-source`) | mapped only: the whole result matches iff the requested code equals the constant, else empty |

Within one param, **comma-separated values OR**; a **repeated** param **ANDs** (standard FHIR). For
`date`, repeats express a range (`recorded-date=ge2021-01-01&recorded-date=le2021-12-31`).

> **camelCase caveat (again):** the query-param parser may rewrite a key's separators, so
> `clinical-status`, `clinical_status`, and `clinicalStatus` are all matched to the same declared
> param by a separator-insensitive normalization (`core/fhir/search.py::_norm`).

### Mapped store — Django ORM

For a mapped resource the specs become ORM filters on the model's own columns (`birthdate` →
`birth_date`, `family` → `name_family`, Observation `date` → `COALESCE(effective_date_time,
effective_period_start)`). `string` uses `__istartswith`; `date` compares at day or instant
precision depending on the column type; `const` short-circuits the whole queryset to empty when the
requested token does not equal the rendered literal. `code`/`identifier` for the *mapped* resources
that already resolve them (Observation `code`, Patient `identifier`) stay in `fhir_search` and are
**not** re-declared in `__search`, so they are never double-applied.

### Auxiliary store — the JSONB query builder

For an aux resource the specs compile to **raw Postgres JSONB** predicates that are attached to the
authorized queryset with `RawSQL` — so the ORM keeps enforcing the patient/practitioner +
organization authorization (as a real queryset, no auth logic duplicated in hand-written SQL) while
the body matching runs as raw SQL. Two mechanisms:

- **Path-membership** (`token`/`identifier`/`code`/`string`/`reference`) compiles to
  `jsonb_path_exists(fhir_data, '<jsonpath>', '<vars-json>')`. The jsonpath runs in **lax mode**, so
  a `.member` step over an array auto-unwraps: **one** predicate matches a `CodeableConcept.coding`
  array, a repeating element, or a scalar alike — and even a doubly-nested path such as
  `participant.role.coding`. Example for `clinical-status=active`:
  `jsonb_path_exists(fhir_data, '$.clinicalStatus.coding ? (@.code == $v)', '{"v":"active"}')`.
- **Date** extracts text with `#>>` and compares lexically — ISO-8601 order **is** chronological
  order — `COALESCE`-ing the polymorphic `[x]` paths:
  `(COALESCE(fhir_data #>> '{effectiveDateTime}', fhir_data #>> '{effectivePeriod,start}')) >= %s`.

Each predicate is annotated as a boolean and `filter`ed on `True`, so multiple params AND naturally
and the annotation is in the `SELECT` list (required under the queryset's `DISTINCT`).

**Injection safety.** No user value is ever interpolated into SQL or into a jsonpath. Every value
reaches Postgres as a **bound parameter**: jsonpath `$vars` for the path-exists predicates,
positional `%s` for the `#>>` date comparisons and `#>>` path arrays. The one place a value becomes
part of a jsonpath — the `like_regex` pattern for `string`/`reference`, which Postgres requires to
be a literal, not a `$var` — is regex-escaped and then escaped as a jsonpath string literal
(`_jsonpath_literal`), and the whole jsonpath is still a bound `%s` parameter, so a crafted value
can neither break out of the pattern nor reach SQL.

### `_sort`

`_sort` takes a comma-separated list of keys, each optionally `-`-prefixed for descending. Two keys
are supported (unknown keys are ignored, per FHIR):

- **`_lastUpdated`** → order by the `last_updated` column (both stores).
- **`date`** → order by the resource's `__sortDate` path(s): mapped → the Django field(s)
  (`COALESCE`d when several); aux → `COALESCE(fhir_data #>> …)` as a `RawSQL` annotation. `__sortDate`
  is the per-resource date path from the US Core sort table (e.g. `Observation` → `effective[x]`,
  `Condition` → `recordedDate`, `MedicationRequest` → `authoredOn`), so `_sort=date` works even for
  resources that do not expose `date` as a *filter*. The sort key is always annotated (not inlined)
  so it is present in the `SELECT` list under `DISTINCT`.

The default order (no `_sort`) remains `-last_updated`.

### `_summary`

`_summary=count` returns just the searchset **total** — a `searchset` Bundle with `total` set,
`entry` empty, and a `self` link — computed with a single `COUNT(*)` over the filtered queryset
before pagination. Other `_summary` values are not specially handled (the full resources are
returned).

## Components

| File                                                                                                                                                                                                                                                               | Responsibility                                                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [core/fhir/fhir_config.json](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/fhir_config.json)                                                                                                                                         | Declares `mapped_resources` (field mappings + `meta.__interaction` / `__criteria`) and `aux_resources` (`resourceType` + `__interaction`), plus each resource's `__search` params and `__sortDate`.       |
| [core/fhir/config.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/config.py)                                                                                                                                                       | Loads the JSON once at import; exposes `get_resource_mapping`, `mapped_interactions` / `aux_interactions`, `mapped_criteria`, `mapped_model_name`, `mapped_search_params` / `aux_search_params`, `mapped_sort_date` / `aux_sort_date`, and **`get_config_errors()`** (validation, see below). |
| [core/fhir/search.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/search.py)                                                                                                                                                       | The US Core search-param, `_sort` and `_summary` layer: mapped ORM filters and the auxiliary Postgres JSONB query builder (`jsonb_path_exists` / `#>>` via `RawSQL`). |
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
returns a **500 OperationOutcome** listing any problems. The checks
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
1. **Every `__search` spec is well-formed**: each entry declares a `type` in
   `token/identifier/code/string/reference/date/const`, a `const` carries a `value`, and every other
   type carries a non-empty `path`.

## Auxiliary resources, FhirSource, `meta.source` & the source header

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

- On every write, the stored body is also **stamped with JHE provenance**
  (`apply_jhe_extensions` in [core/models/fhir_aux_resource.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/models/fhir_aux_resource.py),
  applied by `_persist_aux` in [core/views/fhir.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py))
  so a reader can attribute an opaque aux body to its source and patient without a join:

  - **`meta.source`** is (over)written to `https://jupyterhealth.org/fhir/fhir-source/<id>` — the
    canonical URI of the `FhirSource` the row came through. This is the **authoritative record of
    provenance** and the thing the `_source` read param matches (see below).
  - two patient-attribution `extension` entries (base URL
    `https://jupyterhealth.org/fhir/StructureDefinition/`) carry the patient, because `meta.source`
    names the *source system*, not the patient, and the opaque body may not otherwise resolve to the
    JHE patient:
    - `.../patient-id` — `valueInteger`, the owning patient's pk (from the source);
    - `.../patient-full-name` — `valueString`, the patient's `name_given name_family` (omitted
      entirely when the patient has no name).

  Any prior copies of the JHE extension URLs are stripped first, so re-stamping on update replaces
  the values rather than accumulating duplicates; other extensions on the body are left untouched.
  The `patient_fhir_id` / `fhir_resource_id` columns are computed *before* `_aux_body`
  strips `resourceType`; `meta.source` and the extensions are added *after* `fhir.resources`
  validation, so they never affect the incoming-body check. The same helper is used by the seed
  command so freshly seeded aux rows carry the provenance too.

### FhirSource

A `FhirSource` ([core/models/fhir_source.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/models/fhir_source.py)) is an upstream FHIR source
a **patient registers for themselves** (fields: `patient`, `data_source`, `label`, `fhir_base_url`)
before uploading FHIR resources. CRUD lives at `api/v1/fhir_sources` via
[`FhirSourceViewSet`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir_source.py), scoped to the requesting patient (their `patient`
is assigned server-side).

### The `meta.source` discriminator & the `_source` search param

Every resource JHE serves carries a `meta.source` recording where it came from, and that single
field is the JHE-native-vs-imported discriminator that routes **both reads and writes**:

- **JHE-native (mapped) rows** render a constant `meta.source` = `https://jupyterhealth.org/jhe`,
  declared in the config mapping (a `meta.source` literal on each `mapped_resources` entry). It
  deliberately does **not** live under `/fhir`: these rows are JHE's own system of record projected
  into FHIR, not data that arrived through a FHIR feed.
- **Imported (aux) rows** are stamped `meta.source` = `https://jupyterhealth.org/fhir/fhir-source/<id>`,
  identifying the `FhirSource` (by pk) the row was ingested through.

**Reads choose their store with the standard `_source` search param** (a resource-level `uri` param
matching `meta.source`), so a search is always single-table:

| `_source`                                                     | store                            |
| ------------------------------------------------------------- | -------------------------------- |
| absent                                                        | mapped (JHE-native default)      |
| `https://jupyterhealth.org/jhe`                               | mapped                           |
| `https://jupyterhealth.org/fhir/fhir-source/<id>`             | aux — that one source            |
| `_source:below=https://jupyterhealth.org/fhir/fhir-source/`   | aux — all imported               |
| anything else (an external URI, a typo)                       | empty (no stored `meta.source` matches) |

For a **pure-aux** type (no mapped model, e.g. `Condition`) an absent `_source` means aux — there is
no mapped store to default to. `_source:below` is a **string-prefix** match on the `uri` (FHIR's
`:below` modifier); only the `.../fhir-source/` base is recognized, and it works precisely because
every imported source URI nests under that one prefix. The `X-JHE-FHIR-Source-ID` header plays no
part in reads (it is write-only).

**Why this URI shape:**

- `meta.source` — not a `meta.tag`, a `type` coding, or a custom extension — is the semantically
  correct FHIR element for "where a resource came from"; it is **single-valued** (one row → one
  source → one store, no multiplicity to disambiguate); and `_source` is a **standard**
  all-resource search param, so the routing is idiomatic rather than bespoke.
- Imported URIs are **JHE-minted** (`.../fhir-source/<id>`), not the upstream `fhir_base_url`, so
  they are homogeneous, collision-free (the pk is unique), and share one parent path — which is what
  lets a single `_source:below=.../fhir-source/` mean "everything imported." A raw external base URL
  would be none of those.
- The JHE-native constant sits **outside** the `.../fhir-source/` prefix, so the "all imported"
  `:below` query can never sweep native rows in. (`:below` on the bare `/fhir` or `/jhe` root is *not*
  a supported "native-only" query — the code routes on the exact `.../fhir-source/` prefix.)
- `.../fhir-source/<id>` is an **opaque identifier URI**: nothing is served at that URL, and
  `FhirSource` is intentionally **not** a first-class FHIR resource (FHIR has no such resource type;
  inventing one via a StructureDefinition would be non-conformant). `meta.source` is just a `uri`,
  which the spec permits to be non-dereferenceable.

The constants and parser live in [core/models/fhir_aux_resource.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/models/fhir_aux_resource.py)
(`JHE_NATIVE_SOURCE`, `JHE_FHIR_SOURCE_BASE`, `fhir_source_uri`, `parse_fhir_source_id`); the read
routing is `FHIRResourceView._resolve_source_target` / `_search_source` in
[core/views/fhir.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py).
`JHE_NATIVE_SOURCE` must stay in sync with the `meta.source` literal in
[fhir_config.json](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/fhir/fhir_config.json).

### The `X-JHE-FHIR-Source-ID` header (writes)

A write (create/update/delete) must name the `FhirSource` the new/edited row links to. There are two
ways, in **precedence order** ([resolve_fhir_source_context](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py)):

1. the **`X-JHE-FHIR-Source-ID` header** (the source pk) — **authoritative when present** (it wins
   over the body);
2. the resource body's own **`meta.source`** (`.../fhir-source/<id>`) — the preferred,
   read/write-symmetric way. A **bundle-level `meta.source` is never consulted**: FHIR defines no
   inheritance from a Bundle to its entries, so each entry must carry its own.

Naming **no** source (neither the header nor a parseable body `meta.source`) is **400**. The source
is then resolved to `(patient, fhir_source)`:

- **patient users** are scoped to themselves via the access token; the named source must be theirs;
- **non-patient users** (practitioners) take the patient from the source's `patient`, authorized by
  organization sharing.

An unknown source is **400** and a source the user may not use is **403**. Whichever way the source
was named, the stored `meta.source` is **(over)written to the canonical `.../fhir-source/<id>`** by
`apply_jhe_extensions`, so the persisted provenance is always normalized and trustworthy regardless
of what the client sent.

> **The R4 import endpoint** (`/fhir-import/R4`, [fhir_import.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir_import.py))
> still **requires the header** — it gates the whole request up front (a missing header is a
> request-level **400**, not a per-entry outcome) — since it is a bulk R4→R5 conversion. The
> `meta.source` fallback applies to the native `FHIR/R5` single-create and bundle-batch paths.

## The unified endpoint

A single view, [`FHIRResourceView`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir.py), serves every supported resource at
`FHIR/<version>/<resource>` and `.../<resource>/<id>` (`<version>` is the config `fhir_version`,
e.g. `FHIR/R5/Patient`). It applies
the routing table above, dispatching to the generic **mapped handler** (which translates the
canonical search params into the model's `fhir_search` and renders each row through the config
mapping; `ObservationHandler` subclasses it only for the Base64 serializer and OMH create) or the
**aux handler**. The FHIR bundle batch stays at `POST` on the
base (`FHIR/R5/`), served by [`FHIRBase`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/fhir_base.py), which routes each Observation
entry by the same OMH criteria. Domain and DRF exceptions are rendered as a FHIR `OperationOutcome`
with the right status by `handle_exception`.

## Adding a resource

- **A new aux-only resource**: add `{ "resourceType": "<Name>", "__interaction": ["*"] }` to
  `aux_resources` and restart. No model/serializer/handler changes. To make it searchable/sortable,
  add a `__search` block (param → `{type, path}`, paths being FHIRPaths into the body) and a
  `__sortDate` — no code changes; the JSONB builder is generic.
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
- Routing, single-store `_source` selection, `meta.source` provenance stamping, aux CRUD,
  header-vs-`meta.source` write resolution (header wins), id-shape routing, and the US Core search
  params / `_sort` / `_summary` on both stores (token/identifier/code/string/reference/date filters,
  comma-OR vs repeat-AND, the JSONB builder, date sort, count summary):
  [tests/backend/test_fhir_resource_view.py](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/tests/backend/test_fhir_resource_view.py).
- End-to-end pagination + Bundle validation: `test_patient_pagination` /
  `test_observation_pagination`, validating each page against `fhir.resources.bundle.Bundle`.
