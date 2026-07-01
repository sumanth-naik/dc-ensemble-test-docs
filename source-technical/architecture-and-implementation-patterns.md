---
layout: default
title: Architecture and Implementation Patterns
---

# Salesforce Data Cloud: Architecture and Implementation Patterns

## 1. System Architecture Overview

### Core Components
```
┌─────────────────────────────────────────────────────────────────────┐
│                        DATA CLOUD PLATFORM                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐  ┌──────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │ Ingestion│  │   Identity   │  │ Segmentation│  │ Activation  │  │
│  │  Layer   │──│  Resolution  │──│   Engine    │──│   Gateway   │  │
│  └──────────┘  └──────────────┘  └────────────┘  └─────────────┘  │
│       │              │                  │               │           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Unified Data Model (Lake House)                  │  │
│  │   ┌─────────┐  ┌─────────────┐  ┌──────────────────────┐   │  │
│  │   │  DMOs   │  │ Data Streams │  │  Calculated Insights │   │  │
│  │   └─────────┘  └─────────────┘  └──────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│       │                                                 │           │
│  ┌──────────┐                                    ┌──────────────┐  │
│  │ Metadata │                                    │  Governance  │  │
│  │ Registry │                                    │    Engine    │  │
│  └──────────┘                                    └──────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Infrastructure Layer
- **Compute**: Runs on Salesforce Hyperforce (AWS-based multi-tenant infrastructure)
- **Storage**: Apache Iceberg table format on S3-compatible object storage
- **Processing Engine**: Apache Spark for batch; Apache Flink for streaming
- **Query Engine**: Trino (formerly PrestoSQL) for interactive queries
- **Message Bus**: Apache Kafka for event streaming (managed, multi-tenant)
- **Metadata Store**: Custom catalog service backed by PostgreSQL
- **Cache Layer**: Redis clusters for real-time segment membership lookups

### Data Flow Architecture
1. **Ingestion** → Raw data lands in staging tables (schema-on-read)
2. **Transformation** → Data mapped to DMO schema via declarative mapping rules
3. **Identity Resolution** → Profile unification across sources using configured rulesets
4. **Enrichment** → Calculated insights computed on unified profiles
5. **Segmentation** → Batch and streaming segment evaluation
6. **Activation** → Segment membership pushed to destination systems

## 2. Data Model Design Patterns

### Pattern 1: Hub-and-Spoke (Recommended for Most Implementations)
```
                    ┌──────────────┐
                    │  Individual  │ (Hub - Golden Record)
                    │     DMO      │
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────┴──────┐ ┌─────┴──────┐ ┌─────┴──────┐
    │   Contact   │ │  Account   │ │   Device   │
    │   Point     │ │  (Org)     │ │   Profile  │
    └──────┬──────┘ └─────┬──────┘ └─────┬──────┘
           │               │               │
    ┌──────┴──────┐ ┌─────┴──────┐ ┌─────┴──────┐
    │ Interactions│ │  Products  │ │   Events   │
    │ (Engagement)│ │ (Catalog)  │ │  (Telemetry│
    └─────────────┘ └────────────┘ └────────────┘
```

**When to use**: Standard B2C or B2B2C implementations where individual customers are the primary entity.

**Key design decisions**:
- Individual DMO is the identity resolution target
- All behavioral data links to Individual via foreign key relationships
- Account DMO represents organizational hierarchy (for B2B)
- Interaction events stored in time-series DMOs with timestamp partitioning

### Pattern 2: Multi-Entity Resolution (Complex B2B)
```
    ┌──────────────┐         ┌──────────────┐
    │  Individual  │◄───────►│   Account    │
    │  (Person)    │  M:N    │  (Company)   │
    └──────┬───────┘         └──────┬───────┘
           │                        │
    ┌──────┴──────┐          ┌─────┴──────┐
    │  Contact    │          │  Opportunity│
    │  Points     │          │  History    │
    └─────────────┘          └────────────┘
```

**When to use**: B2B scenarios where both person-level and account-level identity resolution is needed.

**Key design decisions**:
- Two separate identity resolution rulesets (one for Individual, one for Account)
- Junction object manages many-to-many person-to-company relationships
- Segment evaluation can target either Individual or Account entity
- Account hierarchy supports parent/child relationships up to 10 levels

### Pattern 3: Event-Centric (High-Volume Streaming)
```
    ┌──────────────────┐
    │   Event Stream   │ (Primary ingestion point)
    │   (Raw Events)   │
    └────────┬─────────┘
             │ Transform + Enrich
    ┌────────┴─────────┐
    │  Enriched Events │ (Joined with profile attributes)
    └────────┬─────────┘
             │ Aggregate
    ┌────────┴─────────┐
    │   Session DMO    │ (Sessionized behavioral data)
    └────────┬─────────┘
             │ Roll up
    ┌────────┴─────────┐
    │  Individual DMO  │ (Profile with computed metrics)
    └──────────────────┘
```

**When to use**: Media, gaming, or IoT scenarios with very high event volumes (>1M events/hour) where real-time behavioral signals drive activation.

**Key design decisions**:
- Events are first-class citizens, not just interaction history
- Sessionization logic applied at ingestion (configurable timeout: default 30 min)
- Streaming calculated insights update profile metrics in real-time
- Profile is a derived/computed entity, not the primary data store

## 3. Identity Resolution Implementation

### Configuration Parameters
| Parameter | Description | Recommended Value |
|---|---|---|
| `matchRuleTimeout` | Max processing time per batch | 300 seconds |
| `maxGraphSize` | Maximum profiles in a single identity cluster | 50 (prevents over-merging) |
| `confidenceThreshold` | Minimum score for probabilistic match | 0.85 (high precision) |
| `updateFrequency` | How often identity graph recomputes | Every 4 hours |
| `reconciliationStrategy` | Which value survives when records merge | "most_recent" or "source_priority" |
| `unmergeEnabled` | Allow manual or automatic unmerge | true |
| `crossSourceWeight` | Bonus weight for matches across different sources | 1.2x |

### Match Rule Design Best Practices
1. **Start narrow, expand gradually**: Begin with high-confidence deterministic rules (email exact match) before adding probabilistic
2. **Layer rules by confidence**: Rule 1 (email) → Rule 2 (phone + last name) → Rule 3 (address + name fuzzy)
3. **Use normalization**: Always normalize before matching (lowercase, remove whitespace, standardize phone format)
4. **Set appropriate thresholds**: 
   - Email match: confidence 0.99 (near-certain)
   - Phone + name: confidence 0.90
   - Address + name fuzzy: confidence 0.75 (flag for review)
5. **Monitor merge quality**: Review identity clusters > 10 profiles weekly for over-merging
6. **Implement unmerge workflow**: Enable self-service unmerge for customer-facing teams

### Identity Resolution Performance Benchmarks
| Dataset Size | Rule Complexity | Processing Time | Hardware |
|---|---|---|---|
| 1M profiles | 3 deterministic rules | ~8 minutes | Standard |
| 10M profiles | 3 deterministic + 1 probabilistic | ~45 minutes | Standard |
| 50M profiles | 5 deterministic + 2 probabilistic | ~3.5 hours | Large compute |
| 100M profiles | Full ruleset (7 rules) | ~8 hours | Large compute + parallelism |
| 500M profiles | Full ruleset | ~18 hours | Enterprise dedicated |

## 4. Integration Patterns

### Pattern A: Real-Time Event Streaming
```
Web/Mobile App
     │
     ▼ (JavaScript/Mobile SDK)
Data Cloud Web SDK Endpoint
     │
     ▼ (Kafka topic: raw-events)
Stream Processing (Flink)
     │
     ├──▶ Streaming Segment Evaluation
     │         │
     │         ▼
     │    Activation (< 1 minute)
     │
     └──▶ Event Storage (Iceberg)
               │
               ▼
          Batch Processing (scheduled)
```

**Implementation details**:
- Web SDK JavaScript snippet: 2.3KB gzipped, loads asynchronously
- Event schema: flexible (schema-on-read), but typed fields recommended for segmentation
- Throughput: 50K events/second per org (burstable to 200K for 5 minutes)
- Latency: event → segment membership update: median 12 seconds, p99 45 seconds

### Pattern B: Batch File Ingestion
```
Source System
     │
     ▼ (Export to cloud storage)
S3/GCS/Azure Blob
     │
     ▼ (Scheduled connector pull)
Data Cloud Ingestion Service
     │
     ▼ (Schema mapping + validation)
Staging Tables (raw)
     │
     ▼ (Transformation rules)
Target DMO Tables
     │
     ▼ (Triggers identity resolution)
Unified Profile
```

**Implementation details**:
- Supported formats: CSV (with header), Parquet, JSON Lines, Avro
- Max file size: 1GB per file (split larger exports)
- Parallel ingestion: up to 10 concurrent file reads per data stream
- Error handling: bad records written to error table (queryable); threshold configurable (default: reject batch if >5% errors)
- Incremental: supports upsert (match on primary key), append-only, or full-refresh modes

### Pattern C: Zero-Copy Federation (Snowflake Example)
```
Snowflake Account
     │
     ▼ (Secure Data Share / Reader Account)
Data Cloud Zero-Copy Service
     │
     ▼ (Virtual pointer - no data movement)
Virtual DMO (query-time federation)
     │
     ├──▶ Segmentation (pushdown to Snowflake)
     └──▶ Calculated Insights (hybrid execution)
```

**Implementation details**:
- No data physically moves between systems
- Query execution: simple filters pushed down to source; complex joins executed in Data Cloud
- Freshness: reflects source data at query time (no staleness)
- Cost model: Snowflake compute charges apply for pushed-down queries
- Limitations: cannot be used as identity resolution source (must materialize for matching)

### Pattern D: Bidirectional CRM Sync
```
Sales Cloud / Service Cloud
     │
     ▼ (Change Data Capture - CDC)
Data Cloud (real-time sync)
     │
     ▼ (Identity Resolution + Enrichment)
Unified Profile + Calculated Insights
     │
     ▼ (CRM Enrichment Activation)
Sales Cloud / Service Cloud
     (Enriched fields written back)
```

**Implementation details**:
- CDC captures all create/update/delete events on subscribed objects
- Latency: CRM change → Data Cloud reflection: < 2 minutes
- Write-back: segment membership and calculated insight values written to custom fields on Contact/Lead/Account
- Write-back frequency: configurable (real-time, every 15 min, hourly, daily)
- Conflict resolution: Data Cloud values overwrite CRM (one-way enrichment) or merge with timestamp comparison

## 5. Performance Optimization

### Ingestion Optimization
| Technique | Impact | When to Use |
|---|---|---|
| Parquet over CSV | 3-5x faster ingestion | Always (if source supports) |
| Partitioned files | 2x parallel throughput | Files > 100MB |
| Incremental upsert | 10-50x less processing | After initial full load |
| Column pruning at source | Reduces I/O proportionally | When ingesting < 50% of source columns |
| Batch size tuning | Avoid small-file overhead | Combine files < 10MB before upload |

### Query Performance
| Technique | Impact | When to Use |
|---|---|---|
| Pre-compute in Calculated Insights | 100x for repeated queries | Metrics used in multiple segments |
| Limit segment date ranges | Linear improvement | Behavioral segments (last 30/60/90 days) |
| Denormalize frequently joined DMOs | 2-5x for complex segments | Junction objects with high cardinality |
| Use streaming segments for real-time | Eliminates batch wait | Trigger-based activations |
| Partition DMOs by date | Faster time-range queries | Event/interaction tables > 1B rows |

### Identity Resolution Optimization
| Technique | Impact | When to Use |
|---|---|---|
| Reduce max graph size | Prevents runaway clusters | Seeing identity clusters > 100 profiles |
| Pre-normalize data at ingestion | 20-30% faster matching | Inconsistent source data formatting |
| Prioritize deterministic over probabilistic | 5x faster per rule | When deterministic gives > 80% coverage |
| Schedule during off-peak | More compute available | Non-time-sensitive identity refresh |
| Use blocking keys | 10-100x fewer comparisons | Large datasets (> 50M profiles) |

## 6. Error Handling and Monitoring

### Common Error Patterns and Resolution
| Error | Root Cause | Resolution |
|---|---|---|
| `INGESTION_SCHEMA_MISMATCH` | Source schema changed | Update data stream mapping; re-ingest failed batch |
| `IDENTITY_GRAPH_OVERFLOW` | Cluster exceeded max size | Increase threshold or add disambiguation rules |
| `ACTIVATION_RATE_LIMIT` | Destination API throttling | Enable retry with exponential backoff; increase batch window |
| `CALCULATED_INSIGHT_TIMEOUT` | Query too complex for window | Optimize SQL; break into smaller calculations |
| `CONSENT_BLOCK` | Missing required consent | Expected behavior; verify consent collection coverage |
| `ZERO_COPY_CONNECTION_FAILED` | Source system unavailable | Check partner system status; retry after source recovery |
| `STREAMING_LAG_ALERT` | Consumer falling behind producer | Scale consumer partitions; check for slow downstream |

### Monitoring Dashboard Metrics
- **Ingestion Health**: Records processed/failed per stream per hour; p95 latency
- **Identity Resolution**: Match rate by rule; cluster size distribution; unmerge requests
- **Segmentation**: Refresh duration; membership delta per cycle; evaluation errors
- **Activation**: Push success rate; destination latency; retry count
- **Compute Usage**: Data Services Credits consumed; peak vs. average utilization
