---
layout: bare
title: Feature Catalog and Product Roadmap
---

# Salesforce Data Cloud: Feature Catalog and Product Roadmap

## 1. Current Feature Catalog (Summer '26 Release)

### Identity Resolution
| Feature | Description | Edition |
|---|---|---|
| Deterministic Matching | Exact-match rules on email, phone, loyalty ID | All editions |
| Probabilistic Matching | Fuzzy matching using ML models (name + address + behavioral signals) | Plus, Enterprise |
| Custom Match Rules | Define up to 25 custom rule sets with weighted scoring | Enterprise |
| Identity Graph Visualization | Interactive graph explorer showing merge history and source attribution | Plus, Enterprise |
| Cross-Device Identity | Link anonymous device IDs to known profiles via login events | All editions |
| Reconciliation Rules | Configurable survivorship rules (most recent, most complete, source priority) | All editions |
| Match Rate Analytics | Dashboard showing match rates by source, rule, and confidence band | Plus, Enterprise |

### Segmentation Engine
| Feature | Description | Edition |
|---|---|---|
| Batch Segments | Scheduled refresh (min 1 hour); supports up to 200 segments per data space | All editions |
| Streaming Segments | Real-time evaluation on event arrival; sub-minute membership updates | Plus, Enterprise |
| Waterfall Segments | Prioritized, mutually exclusive segment assignment (e.g., high/med/low value) | All editions |
| Segment on Calculated Insights | Use computed metrics (LTV, propensity scores) as segment criteria | Plus, Enterprise |
| Segment Overlap Analysis | Compare membership across up to 5 segments simultaneously | All editions |
| Segment Trend Tracking | 90-day membership history with daily snapshots | Enterprise |
| Einstein Lookalike Segments | ML-generated expansion audiences from seed segments (min 1K seeds) | Enterprise |

### Data Ingestion
| Feature | Description | Edition |
|---|---|---|
| Native CRM Connector | Zero-copy bidirectional sync with Sales/Service Cloud objects | All editions |
| Cloud Storage Connector | S3, GCS, Azure Blob; CSV, Parquet, JSON formats | All editions |
| Streaming Ingestion (Web SDK) | JavaScript SDK for web event capture; auto-batches at 10 events or 5 seconds | Plus, Enterprise |
| Mobile SDK | iOS/Android native SDKs; offline queue with 72-hour retry | Enterprise |
| Interaction API | Server-side REST API for custom event ingestion; 100K calls/hour/org | All editions |
| MuleSoft Connector | Pre-built connector for Anypoint Platform flows | All editions (requires MuleSoft license) |
| Zero-Copy Partner Network | Snowflake, Databricks, BigQuery, Redshift native shares | Enterprise |
| Google Drive Connector | Ingest documents from Google Shared Drives for unstructured data processing | Plus, Enterprise |
| Salesforce CRM Analytics | Direct data share without ETL | All editions |

### Calculated Insights
| Feature | Description | Edition |
|---|---|---|
| Aggregate Calculations | SUM, COUNT, AVG, MIN, MAX across related records | All editions |
| Window Functions | Rolling 30/60/90/180/365-day windows for trend analysis | Plus, Enterprise |
| Custom SQL Expressions | Write arbitrary SQL for complex business logic | Enterprise |
| Streaming Calculations | Real-time metric updates on event arrival (e.g., running cart total) | Enterprise |
| Einstein Predictions | Pre-built propensity models (purchase, churn, engagement) | Enterprise |
| Recency/Frequency/Monetary | Auto-computed RFM scores with configurable bins | Plus, Enterprise |

### Data Governance
| Feature | Description | Edition |
|---|---|---|
| Consent Management | Granular consent tracking per purpose, channel, and data category | All editions |
| Data Retention Policies | Auto-purge rules by stream, object, or record age | All editions |
| Field-Level Encryption | Customer-managed keys (BYOK) for sensitive attributes | Enterprise |
| Classification Labels | Tag fields as PII, SPI, PHI, Financial with automated policy enforcement | Plus, Enterprise |
| Audit Trail | 2-year immutable log of all data access, segment changes, and activations | All editions |
| Privacy Center Integration | Self-service consumer rights portal (access, delete, portability) | Plus, Enterprise |

### Activation
| Feature | Description | Edition |
|---|---|---|
| Marketing Cloud Activation | Push segments to MC journeys, audiences, and data extensions | All editions |
| Advertising Activation | Google Ads, Meta, TikTok, Snap, Trade Desk, LiveRamp | Plus, Enterprise |
| Commerce Cloud Activation | Real-time segment membership for personalization rules | Enterprise |
| CRM Enrichment | Write-back calculated insights and segment flags to CRM records | All editions |
| Webhook Activation | POST segment changes to any REST endpoint | Plus, Enterprise |
| Tableau Activation | Auto-publish curated datasets for analyst consumption | All editions |
| AppExchange Activation | 30+ partner activation targets via ISV ecosystem | Varies |

## 2. Product Roadmap (Next 3 Releases)

### Winter '27 (Planned)
- **Agentic Data Cloud**: Autonomous agents that can query, segment, and activate on behalf of users using natural language
- **Multi-Org Data Sharing**: Federated identity resolution across up to 5 Salesforce orgs without data movement
- **Vector Embeddings**: Native embedding generation for unstructured data (documents, support transcripts) to power semantic search
- **Streaming Joins**: Real-time enrichment of streaming events with profile attributes at sub-100ms latency

### Spring '27 (Planned)
- **Data Cloud for Industries**: Pre-built DMOs and identity rules for 8 industry verticals
- **Graph Analytics**: Relationship-aware segmentation (e.g., "customers whose connected contacts also purchased X")
- **Incremental Calculated Insights**: Process only changed records instead of full recomputation; 10x performance for large datasets
- **External Model Integration**: Bring-your-own ML models (SageMaker, Vertex AI, Azure ML) for custom scoring in Data Cloud

### Summer '27 (Planned)
- **Data Mesh Architecture**: Domain-driven data spaces with federated governance and self-serve provisioning
- **Event-Sourced DMOs**: Full event history with point-in-time profile reconstruction
- **Data Cloud Copilot GA**: Natural language interface for building segments, insights, and troubleshooting data quality

## 3. Deprecated Features (Sunset Schedule)

| Feature | Sunset Date | Replacement |
|---|---|---|
| Legacy CDP (Interaction Studio) | March 2027 | Data Cloud + Personalization |
| Salesforce DMP (Krux) | Already sunset | Data Cloud Advertising Activation |
| Datorama Reports Classic | June 2027 | Marketing Cloud Intelligence |
| Customer 360 Audiences (V1) | December 2026 | Data Cloud Segmentation Engine |

## 4. Licensing and Edition Comparison

| Capability | Data Cloud Starter | Data Cloud Plus | Data Cloud Enterprise |
|---|---|---|---|
| Profiles included | 10K | 100K | 1M+ (custom) |
| Data Services Credits | 250K/month | 2.5M/month | Custom |
| Identity Resolution | Deterministic only | Deterministic + Probabilistic | Full (+ custom rules) |
| Streaming | Not included | Web SDK | Web + Mobile + Interaction API |
| Zero-Copy Partners | Not included | 1 partner | Unlimited |
| Calculated Insights | Basic aggregates | + Window functions | + Custom SQL + Streaming |
| Activations | CRM + MC only | + Advertising + Tableau | + Commerce + Webhooks |
| Support | Standard | Premier | Signature |
| Price (list) | $65K/year | $175K/year | Custom (typically $300K+) |
