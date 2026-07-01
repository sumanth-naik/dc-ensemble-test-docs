---
layout: default
title: Use Cases and Business Requirements
---

# Salesforce Data Cloud: Use Cases and Business Requirements

## 1. Industry Use Cases

### Retail & E-Commerce
- **Unified Customer Profiles**: Consolidate POS, web, mobile app, and loyalty program data into a single Golden Record per customer
- **Real-Time Personalization**: Trigger product recommendations within 200ms using streaming segments
- **Inventory-Aware Marketing**: Suppress promotions for out-of-stock items by ingesting inventory feeds every 15 minutes
- **Cross-Channel Attribution**: Attribute conversions across email, paid social, organic search, and in-store using Data Cloud's identity resolution with a 94.7% match rate (measured across 12M profiles in pilot)

### Financial Services
- **Next-Best-Action for Advisors**: Surface propensity-scored offers on the advisor console using calculated insights refreshed every 4 hours
- **AML Transaction Monitoring**: Stream transaction events (avg 3.2M/day per mid-tier bank) through real-time rules before batch reconciliation
- **Client 360 for Wealth Management**: Unify custodial accounts, CRM interactions, and external market data; identity graph resolves across 5+ ID namespaces (SSN hash, account #, email, phone, device ID)
- **Regulatory Reporting**: Pre-aggregate Data Cloud segments into regulatory cohorts (Basel III, MiFID II) with full lineage tracking

### Healthcare & Life Sciences
- **Patient Journey Orchestration**: Map touchpoints from scheduling through post-discharge follow-up; median journey spans 47 days and 12 touchpoints
- **HCP Engagement Scoring**: Score physician engagement combining rep visits, sample requests, webinar attendance, and publication co-authorship
- **Clinical Trial Matching**: Cross-reference EHR diagnosis codes (ICD-10) with active trial inclusion criteria stored in Data Cloud custom objects
- **Consent & Preference Management**: Track HIPAA authorization and marketing opt-in/out per channel per data category

### Manufacturing
- **Dealer/Distributor Intelligence**: Unify dealer CRM data with warranty claims and IoT telemetry to predict dealer performance quartile
- **Connected Product Insights**: Ingest IoT sensor data (temperature, vibration, usage hours) at up to 50K events/sec for predictive maintenance triggers
- **Supply Chain Visibility**: Correlate purchase orders, shipment tracking, and demand forecasts in a unified Supply Chain DMO

## 2. Business Requirements by Capability

### Segmentation Requirements
| Requirement | Specification |
|---|---|
| Segment refresh frequency | Batch: every 1-12 hours; Streaming: sub-minute |
| Max segment size | Tested up to 850M profiles in a single segment |
| Nested segment support | Up to 5 levels of nested segment logic |
| Cross-object segmentation | Supported across all related DMOs within a data space |
| Exclusion segments | Boolean NOT logic; supports up to 10 exclusion criteria per segment |
| Lookalike modeling | Built-in via Einstein; requires minimum 1,000 seed profiles |

### Activation Requirements
| Channel | Latency SLA | Volume Tested |
|---|---|---|
| Marketing Cloud (email) | < 5 minutes | 120M contacts/activation |
| Google Ads (audience) | < 60 minutes | 45M profiles/sync |
| Meta Custom Audiences | < 60 minutes | 80M profiles/sync |
| Commerce Cloud (recs) | < 200ms (streaming) | 2.3M rec requests/hour |
| Tableau (analytics) | Same-day | 500M row datasets |
| Salesforce CRM (enrichment) | < 15 minutes | 30M contact updates/batch |

### Data Ingestion Requirements
| Source Type | Protocol | Refresh Cadence |
|---|---|---|
| Salesforce CRM objects | Native connector (zero-copy) | Real-time CDC |
| Cloud storage (S3/GCS/ADLS) | Bulk file ingestion | Every 15 min - 24 hours |
| Streaming events (web/mobile) | Web SDK / Mobile SDK / Interaction API | Real-time (< 1 sec) |
| MuleSoft integrations | MuleSoft connector | Per-flow schedule |
| Snowflake/Databricks/BigQuery | Zero-copy partner network | Federated (no move) |
| Legacy databases | Ingestion API (batch) | Hourly-daily |

## 3. ROI Metrics and Business Outcomes

### Measured Results (Anonymized Customer Data)
- **Retailer A** (150 stores): 23% increase in repeat purchase rate within 90 days of Data Cloud deployment; attributed to unified profiles enabling cross-channel retargeting
- **Bank B** (8M customers): $4.2M annual savings from decommissioning 3 legacy CDPs; additional $1.8M from reduced manual data reconciliation
- **Healthcare System C** (12 hospitals): 31% reduction in patient no-shows through journey orchestration triggered by Data Cloud segments
- **Manufacturer D**: 18% improvement in dealer satisfaction scores after providing dealers with connected-product insights powered by IoT data in Data Cloud

### Total Cost of Ownership Considerations
- License cost: varies by Data Cloud edition (Starter, Plus, Enterprise)
- Implementation: typically 8-16 weeks for first use case; 3-6 months for enterprise rollout
- Data storage: included credits vary by edition; overages billed per TB/month
- Compute: segment refresh and identity resolution consume "Data Services Credits" — budget 2-5x your profile count per month
- Integration: MuleSoft licenses separate; native connectors included

## 4. Compliance and Regulatory Requirements

### GDPR
- Data residency in EU regions (Frankfurt, Dublin)
- Right to erasure: Data Cloud supports cascading delete across all DMOs within 72 hours of request
- Consent enforcement: segments automatically exclude profiles lacking required consent categories
- Data minimization: configurable retention policies per data stream (30 days - unlimited)

### CCPA/CPRA
- Do-Not-Sell flag honored in all activation targets
- Consumer data access requests serviced via Privacy Center integration
- Sensitive Personal Information (SPI) classification supported at field level

### HIPAA
- BAA available for Healthcare edition
- PHI fields encrypted at rest (AES-256) and in transit (TLS 1.3)
- Audit trail: 2-year retention of all data access and segment membership changes
- Minimum Necessary Rule: field-level access controls on DMO attributes

### Industry-Specific
- PCI-DSS: tokenized card data ingestion; raw PANs never stored in Data Cloud
- SOX: calculated insights used for financial reporting have full formula audit trail
- FERPA: Education Cloud integration respects student data consent boundaries
