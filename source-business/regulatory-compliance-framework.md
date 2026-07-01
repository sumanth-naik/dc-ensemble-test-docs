---
layout: default
title: Regulatory Compliance Framework
---

# Salesforce Data Cloud: Regulatory Compliance Framework

## 1. Data Residency and Sovereignty

### Available Regions
| Region | Data Center Locations | Regulations Served |
|---|---|---|
| North America (NA) | Virginia (US-East), Oregon (US-West), Montreal (Canada) | CCPA, CPRA, PIPEDA, SOX |
| Europe (EU) | Frankfurt (Germany), Dublin (Ireland) | GDPR, ePrivacy, DORA |
| Asia Pacific (APAC) | Tokyo (Japan), Sydney (Australia), Mumbai (India) | APPI, Privacy Act 1988, DPDPA |
| United Kingdom (UK) | London | UK GDPR, DPA 2018 |
| Middle East (ME) | UAE (coming Winter '27) | PDPL (Saudi Arabia) |

### Data Residency Guarantees
- All profile data, segments, and calculated insights stored within the selected region
- Identity resolution processing occurs within the same region (no cross-border data movement for matching)
- Metadata (schema definitions, configuration) may be stored in the org's home region
- Backup and disaster recovery replicas stay within the same geographical boundary
- Cross-region activation requires explicit data transfer agreement configuration

## 2. Consent Management Architecture

### Consent Data Model
```
Individual (DMO)
  └── Consent Records
        ├── Purpose (e.g., "Marketing", "Analytics", "Personalization")
        ├── Channel (e.g., "Email", "SMS", "Push", "Advertising")
        ├── Status (Opted-In | Opted-Out | Not Provided)
        ├── Legal Basis (Consent | Legitimate Interest | Contract | Legal Obligation)
        ├── Capture Timestamp (ISO 8601)
        ├── Expiry Date (auto-enforce)
        ├── Source (web form, preference center, import, verbal)
        └── Version (links to specific T&C version accepted)
```

### Consent Enforcement Rules
1. **Pre-Segmentation Filter**: Profiles without required consent are excluded BEFORE segment evaluation (not after)
2. **Activation Gate**: Each activation target maps to one or more consent purposes; activation blocked if consent missing
3. **Consent Decay**: Configurable auto-expiry (default: 24 months for marketing consent in EU)
4. **Double Opt-In**: Supported for email; pending consent status until confirmation click
5. **Preference Cascade**: Org-level opt-out overrides channel-specific opt-ins
6. **Consent Audit**: Every consent state change logged with full before/after snapshot

### Consent API Integration
- **Preference Center SDK**: Embeddable widget for web/mobile; syncs to Data Cloud in real-time
- **Consent Ingestion API**: Batch import consent records from external systems (OneTrust, TrustArc, Ketch)
- **Consent Query API**: Check consent status before external system activation (for custom integrations)
- **Salesforce CRM Sync**: Bi-directional consent sync with Contact/Lead IndividualConsent records

## 3. Data Subject Rights Implementation

### Right to Access (GDPR Art. 15, CCPA Sec. 1798.100)
| Step | System Action | SLA |
|---|---|---|
| Request received | Privacy Center creates case + triggers Data Cloud export job | Immediate |
| Profile assembly | Identity graph resolves all linked records across DMOs | < 1 hour |
| Data compilation | All attributes, segment memberships, consent records, and interaction history exported | < 4 hours |
| Format delivery | JSON + CSV dual-format download link sent to requestor | < 24 hours |
| Verification | Request authenticated via email challenge or identity verification flow | Before processing |

### Right to Erasure (GDPR Art. 17, CCPA Sec. 1798.105)
| Step | System Action | SLA |
|---|---|---|
| Request received | Privacy Center creates case + flags profile for deletion | Immediate |
| Identity resolution | All linked profiles in identity graph identified for cascading delete | < 1 hour |
| Suppression | Profile immediately excluded from all active segments and activations | < 15 minutes |
| Hard delete - profiles | Profile records purged from all DMOs | < 48 hours |
| Hard delete - events | Interaction history permanently removed | < 72 hours |
| Hard delete - backups | Removed from backup systems on next rotation cycle | < 30 days |
| Confirmation | Deletion certificate generated with record counts and timestamps | After completion |
| Downstream notification | Connected systems notified to delete via webhook/API | Concurrent with profile delete |

### Right to Portability (GDPR Art. 20)
- Export formats: JSON-LD (structured), CSV (tabular), or machine-readable XML
- Includes: all data provided by the data subject + derived data from automated processing
- Excludes: aggregated/anonymized data that cannot be attributed to individual
- Transfer: direct transmission to another controller supported via secure API endpoint

### Right to Rectification (GDPR Art. 16)
- Self-service correction via Privacy Center portal
- Corrections propagate across all unified profile records within 1 segment refresh cycle
- Survivorship rules may need manual override for corrected fields (admin action)

## 4. Audit and Compliance Reporting

### Audit Trail Coverage
| Event Category | Retention | Detail Level |
|---|---|---|
| Data access (queries) | 2 years | User, timestamp, object, fields accessed, record count |
| Segment membership changes | 2 years | Profile ID, segment, action (add/remove), trigger |
| Activation events | 2 years | Segment, target, volume, status, error details |
| Configuration changes | 2 years | User, object, field, old value, new value |
| Consent changes | Indefinite | Full before/after with source attribution |
| Identity resolution events | 1 year | Merge/unmerge actions, rule matched, confidence score |
| Admin actions | 2 years | Login, permission changes, data stream modifications |

### Compliance Reports (Pre-Built)
1. **Data Inventory Report**: All data streams, objects, and fields with classification labels and retention policies
2. **Consent Status Report**: Current consent coverage by purpose, channel, and region
3. **Data Subject Request Report**: Open/closed requests with SLA compliance metrics
4. **Cross-Border Transfer Report**: All data movements between regions with legal basis
5. **Retention Compliance Report**: Records approaching or exceeding retention limits
6. **Access Control Report**: Who has access to what data, with last-access timestamps
7. **Activation Compliance Report**: All audience pushes with consent validation status

### Certifications and Attestations
| Certification | Scope | Renewal Cycle |
|---|---|---|
| SOC 2 Type II | Security, Availability, Confidentiality | Annual |
| ISO 27001 | Information Security Management | 3-year (annual surveillance) |
| ISO 27701 | Privacy Information Management | 3-year (annual surveillance) |
| ISO 27018 | PII Protection in Cloud | 3-year (annual surveillance) |
| HIPAA | Healthcare data handling | Annual attestation (not certification) |
| PCI-DSS Level 1 | Payment card data | Annual |
| FedRAMP (Moderate) | US Government cloud use | Annual |
| C5 (Germany) | Cloud computing compliance | Annual |
| IRAP (Australia) | Government security assessment | Biennial |
| TX-RAMP | Texas government cloud use | Annual |

## 5. Data Processing Agreements and Controls

### Standard DPA Terms
- **Sub-processor list**: Published and updated 30 days before new sub-processor engagement
- **Data breach notification**: Within 72 hours of confirmed breach (aligns with GDPR Art. 33)
- **Data deletion on termination**: All customer data purged within 90 days of contract end
- **Right to audit**: Customer may audit Salesforce data processing once per year with 30-day notice

### Technical Controls
| Control | Implementation |
|---|---|
| Encryption at rest | AES-256; customer-managed keys available (BYOK/HYOK) |
| Encryption in transit | TLS 1.3 (TLS 1.2 minimum); certificate pinning for mobile SDKs |
| Access control | Role-based (RBAC) + attribute-based (ABAC) policies |
| Network isolation | Dedicated Hyperforce infrastructure; VPC peering available for zero-copy |
| Key management | HSM-backed key storage; automatic rotation every 365 days (configurable) |
| Tokenization | Payment card and SSN tokenization at ingestion; reversible only with specific permission |
| Data masking | Dynamic masking rules per user role (full, partial, hashed, null) |
| Pseudonymization | Automatic pseudonymization for analytics workloads; re-identification requires separate key |

## 6. Industry-Specific Compliance

### Healthcare (HIPAA)
- BAA (Business Associate Agreement) required and available
- PHI minimum necessary access enforced via field-level security
- Patient matching uses HIPAA-safe harbor de-identification method for probabilistic rules
- Break-glass access logging for emergency clinical data access
- Automatic PHI field detection suggests classification during data stream setup

### Financial Services (SOX, GLBA, DORA)
- Calculated Insights formula audit trail for SOX compliance
- Material financial data flagged for enhanced change management
- DORA (Digital Operational Resilience Act): ICT risk reporting templates pre-built
- GLBA: financial privacy notice tracking integrated with consent management

### Education (FERPA)
- Student data isolation at data space level
- Directory information vs. education records classification enforced
- Parental consent tracking for minors (configurable age threshold)
