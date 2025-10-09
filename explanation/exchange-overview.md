---
title: Overview
---

JupyterHealth Exchange is a Django web application that facilitates the sharing of user-consented medical data with authorized consumers through a web UI, REST, and FHIR APIs.

In the context of JupyterHealth, data producers are typically study participants (FHIR *Patients*) using the [CommonHealth Android App](https://play.google.com/store/apps/details?id=org.thecommonsproject.android.phr) linked to personal devices (e.g., Glucose Monitors), and data consumers are typically researchers (FHIR *Practitioners*).

```{image} ../assets/images/jupyterhealth-exchange-overview.jpg
---
alt: Diagram of application components
width: 800px
---
```

## Features

- OAuth 2.0, OIDC, and SMART on FHIR Identity Provision using [django-oauth-toolkit](https://github.com/jazzband/django-oauth-toolkit)
- FHIR R5 schema validation using [fhir.resources](https://github.com/glichtner/fhir.resources)
- REST APIs using [Django Rest Framework](https://github.com/encode/django-rest-framework)
- Built-in, lightweight Vanilla JS SPA UI (npm not required) using [oidc-client-ts](https://github.com/authts/oidc-client-ts), [handlebars](https://github.com/handlebars-lang/handlebars.js), and [bootstrap](https://github.com/twbs/bootstrap)

## Status

This project is currently in a Proof of Concept stage. You can monitor progress in our [GitHub Project](https://github.com/orgs/the-commons-project/projects/8).
