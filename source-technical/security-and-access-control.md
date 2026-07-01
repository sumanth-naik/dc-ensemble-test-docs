---
layout: default
title: Security Implementation and Access Control
---

# Salesforce Data Cloud: Security Implementation and Access Control

## 1. Authentication and Authorization Architecture

### Authentication Layers
```
┌─────────────────────────────────────────────────────────┐
│                    Access Request                         │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 1: Salesforce Platform Authentication             │
│  (SSO/SAML/OAuth 2.0/MFA)                               │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 2: Data Cloud Permission Sets                     │
│  (Admin/Specialist/Viewer/Custom)                        │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 3: Data Space Access Control                      │
│  (Org-level isolation of data domains)                   │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 4: Object/Field-Level Security                    │
│  (DMO and field visibility per profile/permission set)   │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Layer 5: Record-Level Security (Data Policies)          │
│  (Row-level filtering based on user attributes)          │
└─────────────────────────────────────────────────────────┘
```

### OAuth 2.0 Configuration for External Connectors
```yaml
Named Credential Configuration:
  name: "DataCloud_GoogleDrive_Prod"
  authentication_protocol: OAuth 2.0
  authentication_type: "Named Principal"  # or "Per User"
  
  OAuth Settings:
    authorization_endpoint: "https://accounts.google.com/o/oauth2/v2/auth"
    token_endpoint: "https://oauth2.googleapis.com/token"
    scope: 
      - "https://www.googleapis.com/auth/drive.readonly"
      - "https://www.googleapis.com/auth/drive.labels.readonly"
    grant_type: "authorization_code"
    callback_url: "https://{myDomain}.my.salesforce.com/services/authcallback/{authProviderId}"
    
  Token Management:
    access_token_expiry: 3600 seconds
    refresh_token_rotation: enabled
    token_storage: encrypted (platform credential store)
    
  Security:
    require_pkce: true
    state_parameter: auto-generated (CSRF protection)
    token_encryption_at_rest: AES-256
```

### API Authentication for Data Cloud APIs
```yaml
Connect API Authentication:
  base_url: "https://{myDomain}.my.salesforce.com/services/data/v{version}/ssot/"
  
  Methods:
    1. Session-based (UI context):
       header: "Authorization: Bearer {sessionId}"
       
    2. OAuth 2.0 (Server-to-Server):
       grant_type: "client_credentials"  # or "jwt-bearer" for service accounts
       token_endpoint: "https://login.salesforce.com/services/oauth2/token"
       scopes: ["cdp_api", "cdp_ingest_api", "cdp_segment_api"]
       
    3. Connected App (automated processes):
       consumer_key: "{connected_app_consumer_key}"
       private_key: RSA-2048 (for JWT bearer flow)
       subject: "{integration_user_username}"
       audience: "https://login.salesforce.com"
       
  Rate Limits:
    - Query API: 10 requests/second per user; 100/second per org
    - Ingest API: 100K records/hour per org (burstable)
    - Segment API: 50 requests/second per org
    - Profile API: 200 requests/second per org
```

## 2. Data Space Isolation

### Multi-Tenant Isolation Model
```
Organization (Org)
  └── Data Space 1: "Marketing" 
  │     ├── Data Streams (marketing sources only)
  │     ├── DMOs (marketing-specific objects)
  │     ├── Segments (marketing audiences)
  │     ├── Activations (marketing channels)
  │     └── Users: Marketing team members
  │
  └── Data Space 2: "Sales Analytics"
  │     ├── Data Streams (CRM + revenue data)
  │     ├── DMOs (sales-focused objects)
  │     ├── Segments (sales-relevant cohorts)
  │     ├── Activations (CRM enrichment only)
  │     └── Users: Sales ops + RevOps team
  │
  └── Data Space 3: "Customer Service"
        ├── Data Streams (case data + CSAT)
        ├── DMOs (service-specific objects)
        ├── Segments (service cohorts)
        ├── Activations (routing + escalation)
        └── Users: Service analytics team
```

### Data Space Security Properties
| Property | Behavior |
|---|---|
| Data visibility | Users see ONLY data in their assigned data space(s) |
| Identity resolution | Runs within data space boundaries (no cross-space merging by default) |
| Segment scope | Segments can only reference DMOs within their data space |
| Activation | Activations can only use segments from their data space |
| Cross-space sharing | Requires explicit "Shared DMO" configuration (admin-only) |
| Admin visibility | Data Cloud Admins can see all data spaces |
| Audit | Actions logged with data space context for compliance |

### Cross-Data-Space Access Pattern
```yaml
Shared DMO Configuration:
  shared_object: "Individual" (Golden Record)
  source_data_space: "Marketing"
  target_data_spaces: ["Sales Analytics", "Customer Service"]
  shared_fields:  # Explicitly whitelist fields for sharing
    - unified_individual_id
    - first_name
    - last_name
    - email_address
    - lifetime_value_tier  # Enum, not exact value
  excluded_fields:  # Never share these across spaces
    - raw_behavioral_events
    - marketing_consent_details
    - sensitive_health_indicators
  access_type: "read_only"  # Target spaces cannot modify shared data
```

## 3. Encryption Implementation

### Encryption at Rest
| Data Type | Encryption Method | Key Management |
|---|---|---|
| Profile data (DMOs) | AES-256-GCM | Salesforce-managed (default) or BYOK |
| Event data (streaming) | AES-256-GCM | Salesforce-managed |
| Calculated Insights | AES-256-GCM | Inherits from source DMO |
| Configuration metadata | AES-256 | Salesforce-managed (non-configurable) |
| Audit logs | AES-256 | Salesforce-managed (non-configurable) |
| Backup/DR copies | AES-256-GCM | Same key as primary (replicated) |

### Bring Your Own Key (BYOK) Configuration
```yaml
Key Management Setup:
  provider: "AWS KMS" | "Azure Key Vault" | "GCP Cloud KMS" | "Salesforce Key Service"
  
  AWS KMS Configuration:
    key_arn: "arn:aws:kms:us-east-1:123456789:key/abc-def-123"
    key_type: "SYMMETRIC_DEFAULT" (AES-256)
    key_rotation: automatic (annual) or manual
    cross_account_access: via resource policy (Salesforce AWS account granted)
    
  Key Lifecycle:
    creation: customer-initiated via KMS console
    rotation: automatic every 365 days (configurable: 90-365 days)
    revocation: customer can disable key (CAUTION: renders data unreadable)
    destruction: 7-30 day waiting period before permanent deletion
    
  Operational Notes:
    - Key unavailability = Data Cloud reads will fail (5xx errors)
    - Key rotation is online (no downtime; new data encrypted with new key version)
    - Re-encryption of existing data: background process, completes within 72 hours
    - Monitoring: CloudTrail/audit log shows every key usage event
```

### Encryption in Transit
| Connection Type | Protocol | Minimum Version | Certificate |
|---|---|---|---|
| Browser → Salesforce | TLS | 1.2 (1.3 preferred) | Salesforce-managed wildcard |
| API calls | TLS | 1.2 (1.3 preferred) | Salesforce-managed |
| Connector → Source | TLS | 1.2 | Source system's certificate |
| Zero-Copy → Partner | TLS + mTLS | 1.2 | Mutual certificate exchange |
| Internal service mesh | mTLS | 1.3 | Auto-rotated (Istio) |
| Mobile SDK → endpoint | TLS + cert pinning | 1.2 | Pinned public key hash |

## 4. Field-Level Security and Data Classification

### Data Classification Labels
```yaml
Classification Taxonomy:
  PII (Personally Identifiable Information):
    sensitivity: HIGH
    fields: [email, phone, full_name, address, date_of_birth, SSN_hash]
    controls:
      - encryption: mandatory (BYOK recommended)
      - masking: dynamic (show only to authorized roles)
      - retention: maximum 36 months unless legal hold
      - export: requires explicit approval workflow
      - logging: all access logged with user identity
      
  SPI (Sensitive Personal Information - CPRA definition):
    sensitivity: CRITICAL
    fields: [SSN, financial_account, biometric, health_condition, sexual_orientation, race_ethnicity]
    controls:
      - encryption: mandatory (BYOK required)
      - masking: always masked (no role can see raw value in UI)
      - access: separate permission set required ("SPI Access")
      - retention: maximum 12 months
      - export: blocked (API returns hashed value only)
      - logging: enhanced logging with justification required
      
  PHI (Protected Health Information - HIPAA):
    sensitivity: CRITICAL
    fields: [diagnosis_codes, treatment_history, prescription, insurance_id]
    controls:
      - encryption: mandatory (BYOK required)
      - access: "Healthcare Data Access" permission set required
      - BAA: org must have active BAA with Salesforce
      - retention: per state law (varies: 5-10 years)
      - break_glass: emergency access with post-hoc review
      - minimum_necessary: only fields needed for specific role
      
  Financial:
    sensitivity: HIGH  
    fields: [account_balance, transaction_amount, credit_score, income]
    controls:
      - encryption: mandatory
      - access: "Financial Data Access" permission set
      - audit: SOX-relevant; enhanced change tracking
      - retention: 7 years (SOX requirement)
      
  Business Confidential:
    sensitivity: MEDIUM
    fields: [lifetime_value, propensity_score, segment_membership, internal_tier]
    controls:
      - encryption: standard (Salesforce-managed)
      - masking: optional (configurable per role)
      - retention: per business policy
      - export: standard controls
      
  Public:
    sensitivity: LOW
    fields: [company_name, job_title, industry, public_social_profiles]
    controls:
      - encryption: standard
      - masking: none
      - retention: unlimited
      - export: unrestricted
```

### Dynamic Data Masking Rules
```yaml
Masking Policies:
  - policy_name: "PII_Email_Mask"
    applies_to: fields with classification "PII" AND type "email"
    rules:
      - role: "Data Cloud Admin" → show full value
      - role: "Marketing Specialist" → show domain only (***@company.com)
      - role: "Viewer" → show masked (s***h@c***y.com)
      - role: "Analytics" → show hashed (SHA-256 for join operations)
      
  - policy_name: "Financial_Mask"
    applies_to: fields with classification "Financial"
    rules:
      - role: "Finance Admin" → show full value
      - role: "Sales Manager" → show range (tier: $1K-$5K)
      - role: "All others" → hidden (field not visible)
      
  - policy_name: "Phone_Mask"
    applies_to: fields with classification "PII" AND type "phone"
    rules:
      - role: "Service Agent" → show last 4 digits (***-***-1234)
      - role: "Marketing" → hidden
      - role: "Admin" → full value
```

## 5. Network Security

### IP Allowlisting
```yaml
Trusted IP Ranges:
  - name: "Corporate VPN"
    ranges: ["10.0.0.0/8", "172.16.0.0/12"]
    applies_to: ["UI access", "API access"]
    
  - name: "Data Center (Ingestion)"
    ranges: ["203.0.113.0/24"]
    applies_to: ["Ingestion API only"]
    
  - name: "Partner Zero-Copy"
    ranges: ["198.51.100.0/24"]  # Snowflake egress IPs
    applies_to: ["Zero-Copy endpoints"]

Data Cloud Egress IPs (for source system allowlisting):
  NA Region:
    - 52.x.x.x/28 (primary)
    - 54.x.x.x/28 (secondary)
  EU Region:
    - 18.x.x.x/28 (primary)
    - 3.x.x.x/28 (secondary)
  APAC Region:
    - 13.x.x.x/28 (primary)
    - 52.x.x.x/28 (secondary)
```

### Private Connectivity Options
| Option | Use Case | Setup Complexity |
|---|---|---|
| Public Internet + TLS | Default; suitable for most use cases | None |
| Salesforce Private Connect | Dedicated connection to Salesforce via AWS PrivateLink | Medium (requires AWS account) |
| VPC Peering (Zero-Copy) | Direct network path to Snowflake/Databricks | Medium |
| Site-to-Site VPN | Legacy on-premise sources | High (requires network team) |
| MuleSoft Anypoint VPC | Secure MuleSoft → Data Cloud path | Medium (requires MuleSoft Anypoint) |

## 6. Security Monitoring and Incident Response

### Security Event Types
```yaml
Events Monitored:
  Authentication:
    - failed_login (threshold: 5 in 10 minutes → lock account)
    - login_from_new_ip (alert admin)
    - login_from_new_device (require MFA re-challenge)
    - token_refresh_failure (investigate OAuth configuration)
    
  Authorization:
    - permission_elevation (any change to admin permission sets)
    - data_space_access_change (user added/removed from data space)
    - bulk_data_export (> 100K records in single API call)
    - cross_space_query_attempt (denied → log and alert)
    
  Data Operations:
    - schema_modification (DMO field added/removed/type changed)
    - identity_rule_change (match rules modified)
    - retention_policy_change (retention period shortened)
    - activation_target_added (new external system connected)
    - bulk_delete_operation (> 10K records deleted)
    
  System:
    - encryption_key_rotation (BYOK key changed)
    - connector_authentication_failure (source credential invalid)
    - certificate_expiry_warning (30/14/7 days before expiry)
    - anomalous_query_pattern (unusual data access pattern detected by ML)
```

### Incident Response Playbook
```yaml
Severity Levels:
  P1 (Critical - Data Breach Suspected):
    response_time: 15 minutes
    actions:
      1. Isolate: Suspend all activations immediately
      2. Contain: Revoke compromised credentials/tokens
      3. Assess: Determine scope (which profiles, which data)
      4. Notify: Legal team within 1 hour; DPO within 4 hours
      5. Remediate: Patch vulnerability; rotate all keys
      6. Report: Regulatory notification within 72 hours (GDPR)
      
  P2 (High - Unauthorized Access Detected):
    response_time: 1 hour
    actions:
      1. Verify: Confirm unauthorized access via audit trail
      2. Contain: Disable user account; invalidate sessions
      3. Investigate: Determine what data was accessed
      4. Assess: Evaluate if data exfiltration occurred
      5. Remediate: Fix access control gap
      6. Review: Update permission model if systemic issue
      
  P3 (Medium - Policy Violation):
    response_time: 4 hours
    actions:
      1. Log: Document the violation with full audit context
      2. Notify: Inform user's manager and compliance team
      3. Assess: Determine if intentional or accidental
      4. Remediate: Provide retraining or adjust permissions
      5. Monitor: Enhanced monitoring on user for 30 days
```

## 7. Compliance Automation

### Automated Policy Enforcement
```yaml
Policy Engine Rules:
  - rule: "No PII in Advertising Activation"
    condition: activation_target.type IN ("google_ads", "meta", "tiktok")
    enforcement: block_fields_with_classification("PII")
    exception: hashed values allowed (SHA-256 for matching)
    
  - rule: "EU Data Stays in EU"
    condition: individual.residence_country IN (EU_COUNTRIES)
    enforcement: block_activation_to_non_EU_targets
    exception: explicit consent for international transfer (SCCs documented)
    
  - rule: "Consent Required for Email Activation"
    condition: activation_target.channel = "email"
    enforcement: require_consent(purpose="marketing", channel="email")
    action_on_missing: exclude_from_activation (silent, no error)
    
  - rule: "Retention Auto-Purge"
    condition: record.age > data_stream.retention_policy_days
    enforcement: hard_delete (irreversible)
    schedule: daily at 02:00 UTC
    notification: weekly summary to Data Cloud Admin
    
  - rule: "Classification Inheritance"
    condition: calculated_insight derived_from field with classification
    enforcement: inherit_highest_classification_from_source_fields
    example: "If LTV computed from financial data → LTV inherits 'Financial' classification"
```

### Automated Compliance Checks (Weekly)
```yaml
Weekly Compliance Scan:
  checks:
    - name: "Unclassified PII Detection"
      description: "ML scan for fields containing PII patterns without classification label"
      action: flag for manual review; block from activation until classified
      
    - name: "Stale Consent Audit"
      description: "Profiles with consent older than 24 months (EU) or 12 months (configurable)"
      action: mark as "consent_expired"; exclude from all activations
      
    - name: "Orphaned Activation Targets"
      description: "Activation targets with no recent successful push (> 90 days)"
      action: alert admin; recommend decommission
      
    - name: "Permission Drift Detection"
      description: "Users with permissions exceeding their role requirements"
      action: generate access review request for manager approval
      
    - name: "Encryption Coverage"
      description: "Fields with HIGH/CRITICAL classification not using BYOK"
      action: recommend BYOK enrollment; log exception if intentional
```
