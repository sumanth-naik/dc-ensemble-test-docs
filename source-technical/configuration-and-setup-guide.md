---
layout: bare
title: Configuration and Setup Guide
---

# Salesforce Data Cloud: Configuration and Setup Guide

## 1. Initial Provisioning

### Prerequisites Checklist
- [ ] Salesforce org with Data Cloud license provisioned
- [ ] System Administrator profile or custom profile with "Data Cloud Admin" permission set
- [ ] At least one Data Space configured (default created on provisioning)
- [ ] Customer Data Platform enabled in Setup → Data Cloud → Settings
- [ ] Data Cloud Salesforce Connector enabled (for CRM sync)
- [ ] Minimum API version: 58.0 (Summer '23 or later)

### Permission Set Configuration
| Permission Set | Use Case | Key Permissions |
|---|---|---|
| Data Cloud Admin | Full platform administration | All Data Cloud permissions; manage data streams, segments, activations |
| Data Cloud Data Aware Specialist | Data modeling and ingestion | Create/edit DMOs, data streams, mappings; no activation access |
| Data Cloud Marketing Specialist | Segmentation and activation | Create segments, manage activations; no schema modification |
| Data Cloud Viewer | Read-only analytics access | View segments, dashboards, profiles; no modifications |

### Step-by-Step Initial Setup
1. **Enable Data Cloud**: Setup → Feature Settings → Data Cloud → Enable
2. **Create Data Space**: Setup → Data Cloud → Data Spaces → New (name: "Production" or domain-specific)
3. **Configure Salesforce Connector**: Setup → Data Cloud → Salesforce Connector → Map objects
4. **Set up Data Streams**: Setup → Data Cloud → Data Streams → New (choose connector type)
5. **Create Data Model**: Setup → Data Cloud → Data Model → Create/Map DMOs
6. **Configure Identity Resolution**: Setup → Data Cloud → Identity Resolution → Create Ruleset
7. **Create Segments**: Setup → Data Cloud → Segments → New Segment
8. **Set up Activation Targets**: Setup → Data Cloud → Activation Targets → New Target

## 2. Data Stream Configuration

### Salesforce CRM Connector
```yaml
Connection Details:
  Connector Type: Salesforce CRM
  Sync Mode: Real-time (CDC) or Scheduled (batch)
  Objects Available: All standard + custom objects with CDC enabled
  
Configuration:
  refresh_interval: "real-time"  # or "15min", "1hour", "daily"
  sync_deleted_records: true
  include_formula_fields: false  # Not supported in CDC mode
  field_selection: "explicit"    # Only map needed fields to reduce processing
  
Mapped Objects (recommended starting set):
  - Contact → Individual DMO (primary identity source)
  - Account → Account DMO
  - Lead → Individual DMO (with source tagging)
  - Opportunity → Sales Interaction DMO
  - Case → Service Interaction DMO
  - Campaign Member → Engagement DMO
```

### Cloud Storage Connector (S3)
```yaml
Connection Details:
  Connector Type: Amazon S3
  Authentication: IAM Role (recommended) or Access Key
  Bucket: "company-data-exports"
  Path Prefix: "/data-cloud/production/"
  Region: "us-east-1"
  
Configuration:
  file_format: "parquet"          # parquet | csv | json
  compression: "snappy"           # snappy | gzip | none
  schema_detection: "first_file"  # Infer schema from first file encountered
  delimiter: ","                   # For CSV only
  header_row: true                # For CSV only
  null_values: ["", "NULL", "\\N"]
  date_format: "yyyy-MM-dd"
  timestamp_format: "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"
  
Schedule:
  frequency: "every_1_hour"       # every_15_min | every_1_hour | every_6_hours | daily
  start_time: "2024-01-01T00:00:00Z"
  incremental_field: "modified_at"  # Field used to detect new/changed records
  incremental_strategy: "upsert"    # upsert | append | full_refresh
  primary_key: ["customer_id"]
  
Error Handling:
  error_threshold_percent: 5      # Reject entire batch if > 5% records fail
  error_action: "skip_and_log"    # skip_and_log | reject_batch | quarantine
  retry_attempts: 3
  retry_delay_seconds: 60
```

### Streaming Connector (Web SDK)
```yaml
Connection Details:
  Connector Type: Web SDK
  Endpoint: "https://{orgId}.c360a.salesforce.com/api/v1/events"
  
Configuration:
  sdk_version: "2.1.0"
  batch_size: 10                  # Flush after N events
  batch_timeout_ms: 5000          # Flush after 5 seconds regardless
  retry_on_failure: true
  max_retries: 3
  consent_check: "before_send"    # Respect TPC/cookie consent
  anonymous_tracking: true        # Track before identity known
  identity_resolution_event: "login"  # Event that links anonymous → known
  
Schema (example event):
  event_type: "page_view" | "add_to_cart" | "purchase" | "search"
  timestamp: ISO 8601
  session_id: string (auto-generated)
  device_id: string (auto-generated, persisted in localStorage)
  user_id: string (optional, set on login)
  properties:
    page_url: string
    product_id: string
    product_name: string
    category: string
    price: number
    quantity: number
    search_term: string
```

### Google Drive Connector
```yaml
Connection Details:
  Connector Type: Google Drive
  Authentication: OAuth 2.0 (Named Credential)
  Customer ID: "C0xxxxxxx"         # Required since version 1.1.22
  Shared Drive ID: "0AXXXXXXXXXX"

Configuration:
  source_type: "shared_drive"       # shared_drive | my_drive
  file_types: ["pdf", "docx", "txt", "pptx"]
  include_subfolders: true
  max_file_size_mb: 50
  ocr_enabled: true                 # Extract text from images/scanned PDFs
  metadata_labels_enabled: true     # Ingest Google Drive labels as metadata
  refresh_interval: "daily"
  
Processing:
  chunking_strategy: "semantic"     # semantic | fixed_size | page
  chunk_size_tokens: 512
  chunk_overlap_tokens: 50
  embedding_model: "e5-large"
```

## 3. Data Model Object (DMO) Configuration

### Standard DMO Templates
```yaml
Individual DMO (Profile):
  primary_key: unified_individual_id (auto-generated by identity resolution)
  standard_fields:
    - first_name: String(100)
    - last_name: String(100)
    - email_address: Email
    - phone_number: Phone
    - mailing_address: Address (composite)
    - date_of_birth: Date
    - gender: Picklist
    - created_date: DateTime
    - last_modified_date: DateTime
  custom_fields:
    - loyalty_tier: Picklist (Gold, Silver, Bronze, None)
    - lifetime_value: Currency
    - preferred_channel: Picklist (Email, SMS, Push, Mail)
    - acquisition_source: String(255)
    - customer_since: Date
  relationships:
    - has_many: ContactPoint
    - has_many: Engagement
    - belongs_to: Account (optional, for B2B)
    
Contact Point DMO:
  primary_key: contact_point_id
  type_discriminator: contact_point_type (email | phone | address | social)
  standard_fields:
    - value: String(500)
    - type: Picklist
    - is_primary: Boolean
    - is_verified: Boolean
    - consent_status: Picklist (opted_in | opted_out | not_set)
    - created_date: DateTime
  relationships:
    - belongs_to: Individual
    
Engagement DMO:
  primary_key: engagement_id
  partition_key: event_date (date partitioned for query performance)
  standard_fields:
    - event_type: String(100)
    - event_timestamp: DateTime
    - channel: Picklist
    - campaign_id: String(50)
    - content_id: String(50)
    - interaction_value: Number
  relationships:
    - belongs_to: Individual
    - references: Campaign (optional)
```

### Custom DMO Best Practices
1. **Naming convention**: Use PascalCase with domain prefix (e.g., `Retail_Transaction`, `IoT_SensorReading`)
2. **Primary key**: Always define explicit primary key; prefer business keys over surrogate where possible
3. **Partitioning**: Partition time-series DMOs by date (reduces query cost by 10-100x)
4. **Field types**: Use most restrictive type possible (Integer over Float, Date over DateTime if no time needed)
5. **Relationships**: Define relationships at DMO level (not just data mapping) for segmentation support
6. **Cardinality planning**: Document expected record counts; DMOs > 1B rows need partition strategy
7. **Retention**: Set retention policy at DMO creation (hard to change retroactively without re-ingestion)

## 4. Identity Resolution Configuration

### Ruleset Definition
```yaml
Identity Resolution Ruleset: "Production_Customer_Identity"
  Target Entity: Individual DMO
  Update Frequency: Every 4 hours
  Max Identity Cluster Size: 50
  
  Rules (evaluated in priority order):
  
    Rule 1 - Email Exact Match:
      match_fields:
        - field: email_address
          normalization: [lowercase, trim, remove_dots_before_plus]
      match_type: exact
      confidence: 0.99
      
    Rule 2 - Phone + Last Name:
      match_fields:
        - field: phone_number
          normalization: [remove_non_digits, add_country_code]
        - field: last_name
          normalization: [lowercase, trim, remove_accents]
      match_type: exact (both must match)
      confidence: 0.92
      
    Rule 3 - Address + Name Fuzzy:
      match_fields:
        - field: mailing_address.postal_code
          normalization: [remove_spaces]
        - field: last_name
          normalization: [lowercase, soundex]
        - field: first_name
          match_algorithm: jaro_winkler
          threshold: 0.85
      match_type: composite (all must pass)
      confidence: 0.78
      
    Rule 4 - Loyalty ID Cross-Source:
      match_fields:
        - field: loyalty_member_id
          normalization: [uppercase, trim]
      match_type: exact
      confidence: 0.95
      source_requirement: must_cross_sources  # Only match if IDs from different data streams
  
  Reconciliation (Survivorship) Rules:
    default_strategy: most_recent
    field_overrides:
      - email_address: source_priority [CRM, Loyalty, Web, Import]
      - phone_number: most_complete  # Prefer records with country code
      - mailing_address: most_recent
      - date_of_birth: oldest_non_null  # First reported DOB likely most accurate
      - lifetime_value: maximum  # Take highest calculated value
```

## 5. Segment Configuration

### Batch Segment Example
```yaml
Segment: "High_Value_At_Risk_Customers"
  Description: "Customers with LTV > $5000 who haven't engaged in 60+ days"
  Data Space: Production
  Entity: Individual
  Refresh Schedule: Every 6 hours
  
  Criteria:
    ALL of:
      - calculated_insight.lifetime_value > 5000
      - calculated_insight.days_since_last_engagement > 60
      - calculated_insight.days_since_last_engagement <= 180  # Not yet churned
      - consent.email_marketing = "opted_in"
    NONE of:
      - segment_membership includes "Active_Service_Case"
      - individual.do_not_contact = true
      
  Estimated Size: ~45,000 profiles (based on last evaluation)
  
  Activation Targets:
    - Marketing Cloud (re-engagement journey)
    - CRM Enrichment (flag on Contact record for Sales outreach)
```

### Streaming Segment Example
```yaml
Segment: "Cart_Abandonment_Real_Time"
  Description: "Customers who added to cart but haven't purchased within 30 minutes"
  Data Space: Production
  Entity: Individual
  Type: Streaming
  
  Entry Criteria:
    Event: "add_to_cart" received
    AND NOT followed by "purchase" event
    WITHIN: 30 minutes
    AND: cart_value > 50  # Only for meaningful cart values
    AND: consent.push_notification = "opted_in"
    
  Exit Criteria:
    Event: "purchase" received (matching session)
    OR: 24 hours elapsed since entry
    
  Activation:
    Target: Marketing Cloud (triggered send - push notification)
    Delay: 30 minutes after entry (allow natural completion)
    Frequency Cap: Maximum 1 per customer per 7 days
```

## 6. Activation Target Configuration

### Marketing Cloud Activation
```yaml
Activation Target: "SFMC_Production"
  Type: Marketing Cloud
  Connection: Native (same org) or Cross-org (Marketing Cloud Connect)
  
  Mapping:
    segment_membership → Data Extension entry/exit
    individual.email → Subscriber Key (match field)
    individual.first_name → PersonalData.FirstName
    calculated_insight.product_affinity → Attribute: TopCategory
    
  Settings:
    sync_frequency: "every_15_minutes"
    batch_size: 50000
    handle_duplicates: "update_existing"
    on_segment_exit: "remove_from_DE"  # or "mark_inactive"
```

### Advertising Activation (Google Ads)
```yaml
Activation Target: "Google_Ads_Audiences"
  Type: Google Ads
  Authentication: OAuth 2.0 (Named Credential)
  Customer ID: "123-456-7890"  # Google Ads account
  
  Mapping:
    match_keys:  # Used for Customer Match
      - email_address (hashed SHA-256 before send)
      - phone_number (E.164 format, hashed)
      - mailing_address (first name + last name + zip + country)
    
  Settings:
    audience_type: "CUSTOMER_LIST"
    sync_frequency: "daily"
    membership_lifespan_days: 30  # Auto-remove if not refreshed
    consent_signals:
      ad_personalization: required
      ad_user_data: required
```

## 7. Monitoring and Alerting Setup

### Recommended Alert Configuration
```yaml
Alerts:
  - name: "Ingestion Failure"
    condition: data_stream.status = "FAILED"
    threshold: any occurrence
    channel: email + slack
    escalation: after 2 consecutive failures → page on-call
    
  - name: "Identity Resolution Anomaly"  
    condition: match_rate_delta > 10% (compared to 7-day average)
    threshold: any occurrence
    channel: email
    action: pause identity resolution; manual review required
    
  - name: "Segment Size Spike"
    condition: segment.size_change > 50% in single evaluation
    threshold: any occurrence  
    channel: slack
    action: flag for review (possible data quality issue)
    
  - name: "Activation Failure Rate"
    condition: activation.error_rate > 5% in 1-hour window
    threshold: sustained for 2 consecutive hours
    channel: email + pagerduty
    escalation: after 4 hours → escalate to platform team
    
  - name: "Credit Usage Warning"
    condition: monthly_credits_consumed > 80% of allocation
    threshold: any occurrence
    channel: email to admin + finance partner
    action: review usage patterns; consider optimization or license upgrade
```

### Health Check Queries
```sql
-- Ingestion health (last 24 hours)
SELECT data_stream_name, 
       COUNT(*) as total_batches,
       SUM(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END) as successful,
       SUM(records_processed) as total_records,
       SUM(records_failed) as failed_records,
       AVG(processing_duration_seconds) as avg_duration
FROM data_cloud_ingestion_log
WHERE timestamp > CURRENT_TIMESTAMP - INTERVAL 24 HOUR
GROUP BY data_stream_name;

-- Identity resolution quality
SELECT rule_name,
       matches_found,
       match_rate_percent,
       avg_confidence_score,
       clusters_created,
       clusters_merged,
       max_cluster_size
FROM identity_resolution_run_log
WHERE run_date = CURRENT_DATE
ORDER BY rule_priority;

-- Segment freshness
SELECT segment_name,
       last_evaluation_time,
       DATEDIFF(minute, last_evaluation_time, CURRENT_TIMESTAMP) as minutes_since_refresh,
       current_size,
       previous_size,
       (current_size - previous_size) as delta
FROM segment_metadata
WHERE is_active = true
ORDER BY minutes_since_refresh DESC;
```
