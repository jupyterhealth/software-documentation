---
title: Conceptual Overview
description: Learn more about JupyterHealth
---

JupyterHealth Exchange is a Django web application that facilitates the sharing of user-consented medical data with authorized consumers through a web UI, REST, and FHIR APIs.

In the context of JupyterHealth, data producers are typically study participants (FHIR *Patients*) using the [CommonHealth Android App](https://play.google.com/store/apps/details?id=org.thecommonsproject.android.phr) linked to personal devices (e.g., Glucose Monitors), and data consumers are typically researchers (FHIR *Practitioners*).

```{image} ../assets/images/jupyterhealth-exchange-overview.jpg
---
alt: Diagram of application components
width: 800px
---
```

## The Problem: Healthcare Data Silos

Healthcare data today is fragmented across proprietary systems. Wearables, medical devices, and apps each collect valuable health information, but this data remains locked in vendor-specific formats and isolated databases. Researchers struggle to access real-time patient data, and patients have limited control over sharing their own health information.

Traditional healthcare operates on:

- **Proprietary systems** with vendor lock-in
- **Siloed data** that can't easily be shared between systems
- **All-or-nothing consent** where patients must share everything or nothing
- **Delayed insights** due to manual data collection and batch processing

## The Solution: Open Platform for Consent-Based Data Sharing

JupyterHealth Exchange provides a secure, open-source middleware platform that enables:

**For Patients**:

- Granular control over what data to share
- Scope-level consent (e.g., share blood glucose but not weight)
- Study-specific consent (different permissions for different research projects)
- Ability to revoke consent at any time

**For Researchers and Practitioners**:

- Real-time access to consented patient data
- Standardized data formats (FHIR R5, Open mHealth)
- REST and FHIR APIs for programmatic access
- Role-based access control for team collaboration
- Integration with analysis tools (JupyterHub, Python notebooks)

**For Institutions**:

- HIPAA-compliant framework for data management
- Hierarchical organization structure (institution → department → lab)
- Audit trails for data access
- Flexible deployment (cloud or on-premises)

## Part of a Broader Vision

JupyterHealth Exchange is a key component of the [Berkeley Agile Metabolic Health initiative](https://cdss.berkeley.edu/news/berkeley-launches-agile-metabolic-health-and-open-platforms-initiative), a collaboration between UC Berkeley, UCSF, Project Jupyter, 2i2c, and The Commons Project. The initiative aims to transform diabetes and metabolic health treatment by creating an open-source, data-driven healthcare platform.

The goal is to enable:

- **Personalized medicine** based on individual patient data rather than population averages
- **Faster medical innovation** through open data sharing and collaboration
- **Lower healthcare costs** by reducing inefficiencies in data collection
- **Individual empowerment with global reach** where patients control their data while contributing to research

## Who Should Use JupyterHealth Exchange?

### Research Institutions

Conducting clinical trials or observational studies that require:

- Real-time wearable data collection
- Patient consent management
- FHIR-compliant data storage
- Integration with analysis pipelines

### Healthcare Providers

Running patient monitoring programs that need:

- Remote patient monitoring capabilities
- Integration with personal health devices
- Secure data sharing with researchers
- Compliance with healthcare regulations

### Individual Researchers

Working on studies involving:

- Diabetes and metabolic health
- Cardiovascular monitoring
- Sleep and activity tracking
- Digital biomarker development
- Personalized health interventions

### Platform Developers

Building on the JupyterHealth ecosystem to:

- Develop custom data collection apps
- Create analysis dashboards and visualizations
- Integrate new data sources or wearables
- Extend functionality for specific use cases

## What Makes JupyterHealth Exchange Different?

### Granular Consent Management

Unlike all-or-nothing consent models, patients can choose exactly which types of data to share with specific studies. This scope-level consent respects patient autonomy while enabling research.

### Built on Healthcare Standards

- **FHIR R5** for interoperability with healthcare systems
- **Open mHealth (IEEE 1752)** schemas for normalized device data
- **OAuth 2.0/OIDC** for secure authentication
- **SMART on FHIR** OAuth provider for research applications

### Lightweight and Accessible

- Vanilla JavaScript SPA (no complex build tools required)
- Single container deployment option
- Minimal dependencies
- Serves as a reference implementation for others to build upon

### Open Source and Community-Driven

- Developed under the JupyterHub community
- Apache 2.0 license
- Active development on GitHub
- Contributions welcome from researchers, developers, and healthcare professionals

## Core Capabilities

JupyterHealth Exchange provides three main interfaces:

### Web User Interface

A single-page application for practitioners to:

- Create and manage organizations and studies
- Invite and enroll patients
- Configure consent requests
- View patient data and consents
- Generate patient invitation links

### Admin REST API

A comprehensive API for programmatic management:

- Organization hierarchy management
- Study creation and configuration
- Patient enrollment workflows
- Consent management
- Data source configuration

### FHIR API

Standards-based API for healthcare interoperability:

- Patient resource queries
- Observation resource queries (health data)
- FHIR Bundle support for batch operations
- FHIR-compliant search parameters
- Compatible with FHIR tooling and libraries

## Technology Choices

JupyterHealth Exchange makes deliberate technology choices to balance functionality, accessibility, and maintainability:

- **Django**: Mature Python web framework with strong healthcare adoption
- **PostgreSQL**: Rich JSON support for flexible FHIR data storage
- **Vanilla JavaScript**: Avoids npm/webpack complexity for simpler deployment
- **FHIR Resources**: Standard-based data models for healthcare compatibility
- **Open mHealth**: Normalized schemas for wearable and device data

For a deeper understanding of these architectural decisions, see [Architecture](../reference/exchange-architecture.md) in the reference documentation.

## Current Status and Roadmap

**Current State**: Proof of Concept

JupyterHealth Exchange is actively developed and suitable for:

- Research prototypes
- Pilot studies
- Development and testing
- Reference implementation for similar systems

You can monitor progress in our [GitHub Repository](https://github.com/jupyterhealth/jupyterhealth-exchange).

## Getting Started

Ready to explore JupyterHealth Exchange?

**For Developers**: See the [deployment tutorial](../tutorial/exchange-on-kubernetes.md) to set up JupyterHealth Exchange.

**For Understanding**: Continue reading the explanation documentation to understand the technologies and design decisions behind JupyterHealth Exchange.

## Learn More

**External Resources**:

- [JupyterHealth Website](https://jupyterhealth.org/)
- [GitHub Repository](https://github.com/jupyterhealth/jupyterhealth-exchange)
- [Berkeley Agile Metabolic Health Initiative](https://cdss.berkeley.edu/news/berkeley-launches-agile-metabolic-health-and-open-platforms-initiative)

**Explanation Documentation**: For deeper understanding of JupyterHealth Exchange, see:

- [Consent Management](consent-management.md) - How patient consent controls data access
- [Role-Based Access and Governance](rbac-governance.md) - User roles and permissions
- [System Architecture](architecture.md) - Component architecture and deployment
- [Data Flow](data-flow.md) - How data moves through the system
- [Security Overview](security-overview.md) - HIPAA compliance and security controls

## Features

- OAuth 2.0, OIDC, and SMART on FHIR Identity Provision using [django-oauth-toolkit](https://github.com/jazzband/django-oauth-toolkit)
- FHIR schema validation using [fhir.resources](https://github.com/glichtner/fhir.resources)
- REST APIs using [Django Rest Framework](https://github.com/encode/django-rest-framework)
- Built-in, lightweight Vanilla JS SPA UI (npm not required) using [oidc-client-ts](https://github.com/authts/oidc-client-ts), [handlebars](https://github.com/handlebars-lang/handlebars.js), and [bootstrap](https://github.com/twbs/bootstrap)

## Status

This project is currently in a Proof of Concept stage. You can monitor progress in our [GitHub Repository](https://github.com/jupyterhealth/jupyterhealth-exchange).
