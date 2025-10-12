---
title: Security Overview
description: Learn all about JupyterHealth Exchange's security and compliance
---

# Security Overview: Privacy and Compliance

JupyterHealth Exchange (JHE) provides technical capabilities for handling sensitive health data with security controls that support compliance with healthcare privacy regulations including HIPAA (Health Insurance Portability and Accountability Act) and GDPR (General Data Protection Regulation).

## Understanding HIPAA Applicability

```{important}
**JHE Enables HIPAA Compliance, Does Not Enforce It**

The health data flowing through JHE comes from **patient portals via consumer-directed exchange**. While in the patient's control (in the CommonHealth app), this data is **not** Protected Health Information (PHI) under HIPAA.

However, once uploaded to JHE and accessed by research organizations, **HIPAA may apply** depending on:
- Whether the deploying organization is a HIPAA covered entity or business associate
- The nature of the research (e.g., clinical trials may trigger HIPAA)
- Contractual agreements with healthcare providers

**JHE's Role**: Provides the technical safeguards and controls **required** for HIPAA compliance, but **does not automatically make deployments HIPAA-compliant**. Organizations must ensure their deployment, policies, and practices meet HIPAA requirements if applicable to their situation.
```

## Data Classification

**Before Upload to JHE** (in CommonHealth app):
- Patient-controlled personal health information
- Subject to app store privacy policies
- Not PHI under HIPAA (consumer-directed exchange)

**After Upload to JHE** (in research platform):
- Organizational custody and control
- May become PHI if organization is a covered entity/business associate
- Subject to study consent and data use agreements
- Protected by JHE's security controls

**Key Principle**: JHE treats all data with PHI-level security controls regardless of legal classification, enabling organizations to meet HIPAA requirements when necessary.

## Regulatory Framework

### HIPAA-Ready Architecture

JHE implements technical, administrative, and physical safeguards that **support** HIPAA compliance when required:

**Technical Safeguards**:
- **Encryption at rest**: Database encryption for all PHI storage
- **Encryption in transit**: TLS 1.2+ for all API communications
- **Access controls**: Role-based access with organization-level scoping
- **Audit logging**: All data access and modifications are logged
- **Unique user identification**: OAuth 2.0 authentication with individual credentials

**Administrative Safeguards**:
- **Access management**: Organization and study-based authorization
- **Audit controls**: Timestamped consent changes and data access logs
- **Security incident procedures**: Logging for breach detection

**Physical Safeguards**:
- Cloud infrastructure compatible with HIPAA-compliant hosting (typically AWS, Azure, or similar)
- Encrypted backups and disaster recovery procedures

```{note}
**Deployment Responsibility**: JHE provides the technical framework, but organizations must:
- Deploy on HIPAA-compliant infrastructure if required for their use case
- Execute Business Associate Agreements (BAAs) with cloud providers
- Implement organizational policies and training
- Maintain physical security controls
- Conduct risk assessments and security audits
```

### GDPR Compliance

For European patients and research participants, JHE supports GDPR requirements:

**Lawful Basis for Processing**:
- **Consent**: Primary basis - explicit, granular, per-study consent
- **Scientific research**: Secondary basis - legitimate interest for approved research

**Data Subject Rights**:
- **Right to access**: Patients can view their own data and consent status
- **Right to rectification**: Patient records can be updated via API
- **Right to erasure ("right to be forgotten")**: Patient removal from all organizations triggers data deletion
- **Right to data portability**: FHIR API enables data export in standardized format
- **Right to withdraw consent**: Patients can revoke consent at any time

**Data Protection Principles**:
- **Purpose limitation**: Consent is study-specific, not global
- **Data minimization**: Only consented data types are accessible per study
- **Storage limitation**: No automated retention policies yet (manual management required)
- **Integrity and confidentiality**: Encryption, access controls, audit trails

## Authentication and Authorization

### User Types and Access Control

JHE has three distinct user types with different authorization models:

1. **Patient**: Self-access to own data and consent management. Patients always have full control over their own records without requiring roles.
2. **Practitioner**: Organization-scoped access with hierarchical roles (Viewer, Member, Manager)
3. **Super Admin**: System administration with full access (logged and audited)

For detailed information on roles, permissions, governance best practices, and API examples, see [Role-Based Access and Governance](rbac-governance.md).

### Authorization Enforcement

JHE enforces authorization through multiple layers:

1. **Authentication**: Valid OAuth 2.0 token required
2. **User Type Identification**: Patient, Practitioner, or Super Admin
3. **Organization Membership**: Practitioners must belong to organization; patients must be enrolled
4. **Study Enrollment**: Patient must be enrolled in the specific study
5. **Consent Verification**: Patient must have consented to share the data type with that study
6. **Role Permission Check**: Viewer (read-only), Member (patient management), Manager (full admin)

### OAuth 2.0 Flow

JHE uses OAuth 2.0 for authentication:

1. **Client authenticates** via OAuth provider (e.g., CommonHealth app)
2. **Authorization code** returned to client
3. **Access token** obtained via token exchange
4. **API requests** include `Authorization: Bearer {token}` header
5. **Token validation** verifies user identity and type

## Data Protection Mechanisms

### Encryption

**At Rest**:
- Database encryption via PostgreSQL encryption features
- Encrypted backups stored in secure cloud storage
- Environment variables for secrets (never committed to code)

**In Transit**:
- TLS 1.2+ required for all API endpoints
- Certificate pinning in mobile applications
- HTTPS-only policy enforced

### Consent as Authorization

Consent is not just a regulatory checkbox - it's the **primary authorization mechanism**. Every FHIR query checks whether the patient has consented to share the requested data type with the specific study before returning observations:
- No consent record → No data access
- Consent revoked → Future queries blocked
- Consent given → Data accessible to authorized practitioners in that study

### Audit Trail

All consent-related actions are logged:

**Consent Changes**: Each consent decision is recorded with an immutable timestamp, creating a permanent audit trail of when consent was granted or revoked.

**Who Changed What**:
- Patient consent changes: OAuth token identifies the patient
- Practitioner consent changes: OAuth token identifies the practitioner
- Timestamps preserve historical consent state

**Data Access**:
- FHIR API queries are logged (user, study, patient, data types)
- Observation uploads are logged with data source and timestamp
- Failed authorization attempts are logged for security monitoring

## Privacy by Design

### Principle of Least Privilege

- Practitioners only access data within their organization
- Study-level scoping prevents cross-contamination
- Consent is required even for practitioners within the same organization

### Data Minimization

- Only consented data types are returned in FHIR queries
- Observations are filtered at the database level, not application level
- No "bulk data dumps" - queries are scoped to specific studies/patients

### Purpose Limitation

- Consent is study-specific: "Share blood glucose with Diabetes Study"
- Not organization-wide: "Share with University Hospital"
- Not global: "Share everything with everyone"

### Transparency

Patients can view their consent status through the API, which returns all studies requesting consent, the current consent status for each data type in each study, and timestamps of consent changes. This transparency allows patients to understand exactly how their data is being shared.

## Security Considerations for Researchers

### What Researchers Can Access

**With Patient Consent**:
- Observations matching consented data types
- Only for patients enrolled in their study
- Only within their organization

**Cannot Access**:
- Data from other organizations
- Data types not consented
- Patients not enrolled in their studies
- Consent decisions from other studies

### Consent Enforcement

Consent is enforced at **query time**, not upload time. When data is uploaded from the CommonHealth app, it's stored regardless of current consent status. However, when researchers query data, the system checks active consent and only returns observations for which the patient has granted permission.

This design ensures:
- Data is never lost if consent is temporarily revoked
- Researchers cannot access revoked data types
- Patients retain control over data sharing

## Security Monitoring and Incident Response

### What JHE Provides

**Built-in Logging Infrastructure**:
- Django logging framework configured in `jhe/settings.py`
- Authorization failures raise `PermissionDenied` exceptions that can be logged
- Consent changes include `consented_time` timestamps for audit trails
- OAuth 2.0 authentication events logged by Django framework

**Authorization Checks**:
- Multi-layer access control enforced before data access
- Failed consent checks return HTTP 403 Forbidden
- Practitioner role validation at organization level
- Patient self-access vs. practitioner access differentiation

```{important}
**Deployment Responsibility**: JHE provides the logging infrastructure and authorization framework, but organizations must implement their own monitoring, alerting, and incident response procedures appropriate to their deployment environment and compliance requirements.
```

### Recommended Deployment Monitoring

Organizations deploying JHE should implement:

**Application Monitoring**:
- Configure log aggregation (CloudWatch, ELK stack, Splunk, etc.)
- Monitor HTTP 401/403 response rates for unusual patterns
- Track failed authentication attempts
- Alert on super admin access during off-hours
- Review consent change patterns periodically

**Infrastructure Monitoring**:
- Database query performance and anomaly detection
- Network traffic analysis
- Failed login attempt tracking
- Resource utilization metrics

**Incident Response Planning**:
- Document breach notification procedures (HIPAA: 60 days, GDPR: 72 hours)
- Establish credential revocation processes
- Create patient/authority notification templates
- Define roles and escalation procedures
- Test incident response procedures quarterly

## Limitations and Future Enhancements

### Current Limitations

**No Data Retention Policies**:
- Studies remain "active" indefinitely
- No automated data deletion after study completion
- Requires manual removal of patients or studies

**No Built-in Monitoring or Alerting**:
- Django logging framework configured but no alerting system included
- No anomaly detection for unusual access patterns
- No real-time notifications for security events
- Organizations must implement their own monitoring solution

**Limited Audit Logging (HIPAA Compliance Gap)**:
- Consent changes include timestamps (`consented_time`)
- Authorization failures raise exceptions but minimal logging details
- **No per-observation access logging** ("who viewed which data when")
- No built-in audit trail for super admin actions
- No patient-facing access logs ("who has viewed my data")

```{warning}
**HIPAA Requirement**: If your deployment requires HIPAA compliance (i.e., your organization is a covered entity or business associate), you **must** implement detailed access logging per 45 CFR § 164.312(b). This includes:
- User ID of who accessed data
- Date and time of access
- Specific PHI/data accessed (e.g., which observations, which patient)
- Action performed (read, create, update, delete)
- Immutable logs retained for at least 6 years

JHE does not currently provide this functionality. Organizations requiring HIPAA compliance must implement access logging at the application or infrastructure level (e.g., database audit triggers, API gateway logging, custom middleware).
```

## Conclusion

JHE implements security and privacy controls through:
- **Granular consent** as the primary authorization mechanism
- **Multi-layer access controls** at organization, study, and data type levels
- **Encryption and authentication** protecting data at rest and in transit
- **Audit trails** providing accountability and breach detection
- **Patient rights** supporting GDPR and HIPAA requirements

While current implementation provides strong foundational security, ongoing enhancements to audit granularity, automated lifecycle management, and breach detection will further strengthen privacy protections.

## Learn More

- [Role-Based Access and Governance](rbac-governance.md) - Detailed role hierarchy, permissions, and governance best practices
- [Consent Management](consent-management.md) - Detailed consent architecture
- [Data Flow](data-flow.md) - How data moves through the system
- [FHIR Interoperability](fhir-interoperability.md) - API security considerations
