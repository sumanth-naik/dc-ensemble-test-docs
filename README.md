# Salesforce Data Cloud - Ensemble Retriever Test Documents

Test documents for validating Ensemble Retriever across two separate UDLO data sources.

## Structure

```
source-business/       → Source 1 (UDLO #1) — WHAT and WHY
  ├── use-cases-and-business-requirements.md
  ├── feature-catalog-and-roadmap.md
  └── regulatory-compliance-framework.md

source-technical/      → Source 2 (UDLO #2) — HOW
  ├── architecture-and-implementation-patterns.md
  ├── configuration-and-setup-guide.md
  └── security-and-access-control.md
```

## Web Crawler URLs

After pushing to GitHub, use these raw URLs for web crawler ingestion:

**Source 1 (Business):**
- `https://raw.githubusercontent.com/{user}/{repo}/main/source-business/use-cases-and-business-requirements.md`
- `https://raw.githubusercontent.com/{user}/{repo}/main/source-business/feature-catalog-and-roadmap.md`
- `https://raw.githubusercontent.com/{user}/{repo}/main/source-business/regulatory-compliance-framework.md`

**Source 2 (Technical):**
- `https://raw.githubusercontent.com/{user}/{repo}/main/source-technical/architecture-and-implementation-patterns.md`
- `https://raw.githubusercontent.com/{user}/{repo}/main/source-technical/configuration-and-setup-guide.md`
- `https://raw.githubusercontent.com/{user}/{repo}/main/source-technical/security-and-access-control.md`

## Ensemble Test Questions

These questions REQUIRE combining information from BOTH sources to answer fully:

### Cross-Source Questions

1. **"What are the data residency options for GDPR compliance and how is encryption configured for EU deployments?"**
   - Source 1: Compliance doc lists EU regions (Frankfurt, Dublin) and GDPR requirements
   - Source 2: Security doc has encryption implementation details (AES-256, BYOK, TLS 1.3)

2. **"How does identity resolution support the financial services next-best-action use case, and what are the performance benchmarks?"**
   - Source 1: Use cases doc describes financial services next-best-action (propensity scoring, 4-hour refresh)
   - Source 2: Architecture doc has identity resolution performance benchmarks (processing times by dataset size) and configuration parameters

3. **"What streaming capabilities are available in each Data Cloud edition, and how is the streaming architecture actually implemented?"**
   - Source 1: Feature catalog has edition comparison table (Starter=none, Plus=Web SDK, Enterprise=all)
   - Source 2: Architecture doc details streaming infra (Kafka, Flink) and the Web SDK config (batch_size:10, 5s timeout, 50K events/sec)

4. **"What consent management features does Data Cloud provide and how do you technically configure consent enforcement in segments?"**
   - Source 1: Compliance doc has consent data model, enforcement rules, and GDPR consent requirements
   - Source 2: Config guide shows segment configuration with consent criteria (consent.email_marketing = "opted_in") and setup guide has consent API integration details

5. **"What is the ROI for healthcare implementations and what technical setup is needed for HIPAA-compliant patient journey orchestration?"**
   - Source 1: Use cases doc has Healthcare System C ROI (31% reduction in no-shows) and HIPAA requirements
   - Source 2: Security doc has HIPAA technical controls (BAA, PHI classification, break-glass access) and config guide has setup steps

6. **"How does the Google Drive connector work for unstructured data ingestion, and what are the authentication security requirements?"**
   - Source 1: Feature catalog lists Google Drive Connector (Plus/Enterprise editions, for unstructured data)
   - Source 2: Config guide has full connector YAML (customer ID, OAuth, chunking strategy) and security doc has Named Credential OAuth 2.0 setup

7. **"What are the segmentation capabilities and SLAs per edition, and how do you actually configure a streaming segment with frequency caps?"**
   - Source 1: Feature catalog has segmentation features by edition; use cases has activation latency SLAs
   - Source 2: Config guide has actual streaming segment YAML (entry/exit criteria, frequency cap, delay settings)

8. **"What compliance certifications does Data Cloud hold, and how are the underlying technical controls (encryption, access control, monitoring) implemented to maintain those certifications?"**
   - Source 1: Compliance doc lists all certifications (SOC2, ISO 27001, HIPAA, FedRAMP, etc.) with renewal cycles
   - Source 2: Security doc details the technical controls that enable those certifications (encryption specs, RBAC/ABAC, security monitoring events)

9. **"What zero-copy partner integrations are available by edition, and what is the technical architecture and network security setup for zero-copy with Snowflake?"**
   - Source 1: Feature catalog shows zero-copy availability (Enterprise=unlimited, Plus=1 partner)
   - Source 2: Architecture doc has zero-copy federation pattern (virtual DMO, pushdown queries, cost model) and security doc has VPC peering details

10. **"What are the data retention requirements for different regulatory regimes, and how do you configure automated retention policies technically?"**
    - Source 1: Compliance doc has retention requirements per regulation (GDPR=configurable, HIPAA=5-10yr, SOX=7yr)
    - Source 2: Security doc has automated purge policy config (rule definition, daily schedule, notification) and config guide mentions retention at DMO creation
