# Salesforce Data Cloud Architect's Skill

**Source**: *Salesforce Data Cloud Architect's Handbook* (2nd Edition, 2025) by Eliot Harper  
**Purpose**: Reference knowledge base for designing, integrating, and activating customer data solutions using Salesforce Data Cloud.

---

## WHEN TO USE THIS SKILL

Consult this skill whenever:
- Designing or reviewing a Salesforce Data Cloud architecture
- Discussing data ingestion, harmonization, identity resolution, segmentation, or activation
- Making decisions about data modeling, DMO mapping, or schema design
- Advising on best practices for performance, credit consumption, or deployment
- Discussing AI/ML capabilities (Einstein Studio, RAG, vector search) within Data Cloud
- Evaluating integration patterns (BYOL, MuleSoft, Connectors, APIs)

---

## 1. PLATFORM FUNDAMENTALS

### Background & Core Problem
- Salesforce's original platform uses a relational, transactional database not optimized for big data (volume, velocity, variety, veracity, value).
- Big Objects exist but are archival-only; they require duplication to custom objects for operational use.
- Data Cloud was created to store, process, and activate customer data at scale — and to solve interoperability challenges across Salesforce's acquired platforms.

### What Data Cloud Does (High-Level Flow)
1. **Connect** — Ingest data from sources (batch or real-time)
2. **Harmonize** — Map into a standardized canonical data model
3. **Unify** — Use identity graph to resolve individuals across sources
4. **Analyze & Predict** — Calculated insights, segmentation, ML, Tableau
5. **Act** — Activate data to drive engagement across channels

---

## 2. DATA MODEL LAYERS (Medallion Architecture)

| Layer | Object | Role |
|---|---|---|
| Bronze | DSO (Data Source Object) | Raw, native format staging; formulas can be applied |
| Silver | DLO (Data Lake Object) | First inspectable object; Parquet in S3 via Apache Iceberg |
| Gold | DMO (Data Model Object) | Virtual, non-materialized view; used for segmentation/insights |

### Key Concepts
- **Data Source**: Origin system (CRM, Commerce Cloud, S3, SFTP, BYOL, SDK)
- **Data Stream**: Named entity extracted from a Data Source (e.g., "Contacts" from CRM)
- **DSO**: Temporary physical staging store for raw data stream content
- **DLO**: Physical store (Parquet/S3); schema-enforced; categories: Profile, Engagement, Other
- **DMO**: Virtualized grouping of DLO fields; 140+ standard objects; canonical model
- **Data Spaces**: Logical partitions (by brand, region, dept); scope-aware features include Segments, Activations, CIs, Identity Resolution, Data Graphs
- **Data Kits & Packages**: Deployable feature configurations; managed (AppExchange) or unmanaged

### DMO Subject Areas
- Case, Engagement, Loyalty, Party, Privacy, Product, Sales Order

### Category Rules
- **Profile**: Data about individuals or organizations (contact attributes)
- **Engagement**: Time-series data (events); requires an immutable date field
- **Other**: Non-profile, non-engagement (e.g., product catalog, store locations)
- ⚠️ Category cannot be changed after data stream creation

---

## 3. PLATFORM ARCHITECTURE

Built on AWS. Layered microservices:
- Data connectors/ingress: Amazon EKS, Amazon Sync
- Big data processing: Amazon EMR + Apache Spark
- Data storage: S3 (cold/Parquet), DynamoDB (hot), RDS (SQL metadata)
- Data lakehouse: Parquet → Iceberg → Cloud Table → Spark DataFrames → Trino queries

---

## 4. DATA INGRESS

### Web & Mobile SDKs
- **Salesforce Interactions SDK**: Captures web events (profile, engagement, cart, order, consent) into Data Cloud; uses a sitemap + listeners/hooks pattern
- **Engagement Mobile SDK** (formerly MC Mobile SDK): iOS/Android; captures ecommerce, cart, order, custom events

### Salesforce Connectors
- B2C Commerce, Marketing Cloud Engagement (up to 20 DEs per account), Marketing Cloud Personalization (up to 5 datasets), Salesforce CRM (standard + custom objects)
- **Starter Bundles**: Pre-packaged DLO→DMO mappings for common use cases; customizable
- Industry Cloud Starter Bundles available (e.g., Financial Services includes 26 pre-built CIs)

### External Connectors
- **Cloud Storage**: Amazon S3, Azure Storage (Blob + ADLS Gen2), Google Cloud Storage — batch, files up to 200GB, up to 1,000 files/run
- **SFTP**: CSV files; up to 1,000 files/run; SSH key or SSH key + password; PGP encryption supported
- **Amazon Kinesis Connector**: Pull-based; near real-time event/profile data directly from Kinesis streams
- **MuleSoft Anypoint Connector**: Uses Ingestion API + Streaming API; request-response over HTTPS

### Ingestion API
- **Streaming API**: JSON payload; fire-and-forget; async; best for ≤200KB batches; use cases: website signups, order status changes, chatbot events
- **Bulk Ingestion API**: CSV up to 150MB; up to 100 files/job; multi-step (create job → upload → close); best for large regular intervals (daily/weekly)

### BYOL (Bring Your Own Lake) — Zero-ETL Federation
Supported platforms: **Snowflake, Google BigQuery, Databricks, Amazon Redshift**
- "Data in": External entities available as virtual DLOs in Data Cloud (no replication)
- "Data out": Data Cloud objects queryable from BYOL platforms via data shares
- Billable: based on rows processed/accessed
- No credit cost if BYOL platform is in the same AWS region as Data Cloud org
- Acceleration option: periodically persists data in Data Cloud (counted as batch pipeline)

### Data Ingress Best Practices
- Only import data aligned to a specific use case
- Define data governance: understandable, compliant (legal basis for all processing), trusted data
- SFTP: use both SSH key + password; add PGP encryption on top of transport encryption
- Real-time ingestion has cost implications; validate actual use case requirement before implementing
- For real-time, implement async feedback validation (Query API, CIs) since fire-and-forget has no built-in response handling

---

## 5. HARMONIZATION

### Field Mapping
- DLO fields must be mapped to DMOs before they can be used in segmentation/activation
- DLO→DMO is a one-to-one relationship per field (no multiple emails for same individual; split into separate records)
- Data type must match (exception: DLO number → DMO text is allowed)
- After DLO creation, field data types **cannot** be modified
- ⚠️ Validate date formats carefully — auto-detection may be wrong
- DateTime vs Date distinction is critical: date = constant, datetime = subject to timezone conversion

### Fully Qualified Keys (FQK)
- FQK = source key + key qualifier — avoids primary key conflicts across data sources
- Key qualifier fields are auto-created but default to null; must be manually configured
- Always configure for DLO fields with primary or foreign keys mapped to DMO relationships
- Include key qualifier in GROUP BY clauses in Calculated Insights and segment containers

### Formula Fields
- Calculated at ingestion time (not runtime); stored in DLO
- Use cases: static source identifier, math calculations, adding datetime for engagement streams
- Syntax similar to Salesforce formulas but function library differs; not interchangeable

### Data Transforms
Two types:

**Batch Data Transforms** (declarative canvas):
- Nodes: Input, Output, Append, Join (ANSI-style), Aggregate (sum/avg/count/min/max, hierarchical), Filter, Transform, Update
- Input and Output nodes must use matching object types (DLO→DLO or DMO→DMO)
- Billed as Batch Data Pipeline usage

**Streaming Data Transforms** (SQL-based):
- Runs every few minutes; near real-time; reads single records from source DLO → writes to target DLO
- Use cases: splitting denormalized records (phone → Contact Point Phone DLO, email → Contact Point Email DLO), merging fields from different data streams
- ⚠️ DLO key qualifier field cannot be used as source or destination in a transform
- ⚠️ Multiple streaming transforms writing same primary key to same target DLO = last-write-wins
- Billed as Streaming Data Pipeline usage

### Harmonization Best Practices
- Identify system of record for each data source; use formula fields to create composite keys if needed
- Validate DMO category mapping rules (Profile DLOs → Profile or Other DMOs; Profile DLOs cannot map to Engagement DMOs)
- Use standard DMOs first; only extend with custom fields/objects where no standard exists
- Consider ETL pre-processing outside Data Cloud to reduce transform credit consumption

---

## 6. IDENTITY RESOLUTION

### Concepts
- Rulesets based on Individual or Account DMO
- Unified Individual Profile: mutable; changes with data; retains lineage to all source records
- NOT a golden record system — relationships link source individuals to a unified profile; source data is never merged/deleted
- Each individual always has at least one unified individual record, even with no matches
- Maximum: 4 ruleset jobs per data space per 24 hours

### Match Rules
Match criteria types:
- **Exact**: Case-insensitive string comparison
- **Exact Normalized**: Normalizes email (removes delimiters), phone (parses by country code), address (country-specific standardization)
- **Fuzzy**: Probabilistic; uses multilingual DistilBERT AI model; for first name only
  - High Precision: nicknames, international chars (Alexander ↔ Alex)
  - Medium Precision: initials, gender variants (Eliot ↔ Eliott)
  - Low Precision: loose similarities (Liz ↔ Elizabeth)

Match objects available: Contact Point Address, Email, Phone, App, Social; Party Identification

⚠️ Person Accounts from Salesforce CRM cannot be used in identity resolution

### Reconciliation Rules
- Determines preferred value for unified field when conflict exists
- Options: Last Updated, Most Frequent, Source Priority (ranked DLO order)
- Does not apply to contact points (all contact points are retained)
- Can be overridden at field level

### Unified Link Objects
Created per ruleset:
- Unified Link Individual, Account, Contact Point Address, App, Email, Phone, Party Identification
- Can be used in queries and Calculated Insights to debug match rules or validate consolidation rates
- Retain source lineage; unified objects do not

### Party Identifier
- Used to match on Party Identification Type + Identification Name + Party Identification Number
- Party field in Party Identification DMO = Individual Id in Individual DMO
- Use for matching known identifiers (subscriber keys, loyalty IDs) across platforms

### Anonymous to Known Profile Matching
- Anonymous visitors get an Individual ID with session identifier; `Is Anonymous = 1`
- Same session: anonymous profile linked to known Individual on login/purchase via Party Id
- Cross-session: Identity Resolution ruleset unifies session-based individuals with known CRM contacts via match rules

### Identity Resolution Best Practices
- Start with default match rules; add custom rules for high-confidence identifiers (e.g., driver's license, subscriber key)
- Avoid "Match on Blank" — causes over-grouping or excessively large profiles
- Test with sample datasets; create temporary rulesets for permutation testing
- Use Data Explorer + Profile Explorer for validation
- Use this CI query pattern to measure consolidation rate per data source:
```sql
SELECT
  IndividualIdentityLink__dlm.ssot__DataSourceId__c AS DataSourceId__c,
  APPROX_COUNT_DISTINCT(IndividualIdentityLink__dlm.UnifiedRecordId__c) AS Unique_Unified_Individuals__c,
  COUNT(IndividualIdentityLink__dlm.SourceRecordId__c) AS Source_Record_Count__c,
  (1 - APPROX_COUNT_DISTINCT(IndividualIdentityLink__dlm.UnifiedRecordId__c) 
    / COUNT(IndividualIdentityLink__dlm.SourceRecordId__c)) * 100 AS Consolidation_Rate__c
FROM IndividualIdentityLink__dlm
GROUP BY DataSourceId__c, DataSourceObjectId__c
```
- Anonymous profiles don't count toward billable unified profiles, but `Is Anonymous` field must be mapped

---

## 7. INSIGHTS

### Calculated Insights (Batch)
- Derived from scheduled batch data; runs every 6, 12, or 24 hours (or manually)
- Limit: 100,000 CI batch runs/year/tenant (69+ CIs at 6h cadence exceeds this)
- Can query entire data model (all DMOs)
- Result stored in Calculated Insight Object (CIO)
- **Metrics on Metrics**: CIOs can be used in FROM clause of another CI (up to 3 in a chain)
- Measures: COUNT, AVG, SUM, MIN, MAX (up to 5 per CI; number or percentage type)
- Dimensions: up to 10; types: Text, URL, Email, DateTime, Phone, Boolean
- ⚠️ Only measures (not dimensions) can be activated; dimensions can be used as filters

SQL syntax:
```sql
SELECT [DMO fields], [aggregate measures]
FROM [DMO]
[JOIN clauses]
WHERE [optional]
GROUP BY [dimensions]
```

Example (customer spend by product):
```sql
SELECT
  SUM(ssot__SalesOrder__dlm.ssot__GrandTotalAmount__c) AS customer_spend__c,
  ssot__GoodsProduct__dlm.ssot__Name__c AS product__c,
  ssot__Individual__dlm.ssot__Id__c AS custid__c
FROM ssot__GoodsProduct__dlm
JOIN ssot__SalesOrderProduct__dlm 
  ON ssot__SalesOrderProduct__dlm.ssot__ProductId__c = ssot__GoodsProduct__dlm.ssot__Id__c
JOIN ssot__SalesOrder__dlm 
  ON ssot__SalesOrder__dlm.ssot__Id__c = ssot__SalesOrderProduct__dlm.ssot__SalesOrderId__c
JOIN ssot__Individual__dlm 
  ON ssot__Individual__dlm.ssot__Id__c = ssot__SalesOrder__dlm.ssot__SoldToCustomerId__c
GROUP BY custid__c, product__c
```

### Streaming Insights
- Near real-time; derived from SDK/API/Personalization sources only
- Only Engagement DMO can be joined (with Individual and Unified Individual DMOs)
- Measures limited to SUM or COUNT
- Require start/end WINDOW definition (1 minute to 24 hours)
- Used to trigger Data Actions (not usable in Segments or Activations)

```sql
SELECT
  COUNT(ssot__WebSearchEngagement__dlm.ssot__Id__c) AS search_count__c,
  ssot__WebSearchEngagement__dlm.ssot__SearchKeywordsTxt__c AS keywords__c,
  WINDOW.START AS start__c,
  WINDOW.END AS end__c
FROM ssot__WebSearchEngagement__dlm
GROUP BY
  WINDOW(ssot__WebSearchEngagement__dlm.ssot__EngagementDateTm__c, '15 MINUTE'),
  ssot__WebSearchEngagement__dlm.ssot__SearchKeywordsTxt__c
```

### Insights Best Practices
- Use Data Explorer to validate (select "Calculated Insights" for both CI and streaming types)
- Copy SOQL from UI and remove LIMIT 100 for full data validation in Developer Console
- Always include key qualifier fields in JOIN conditions and GROUP BY clauses
- Align refresh schedule to underlying data update frequency; disable unused CIs
- Consider multi-dimensional CIs over multiple individual CIs for same DMO (fewer rows processed)

---

## 8. SEGMENTATION

### Concepts
- Segments target entities: Account, Account Contact, Individual, Unified Individual, Lead, User (profile category only)
- Filters: Direct Attributes (one-to-one) and Related Attributes (one-to-many, create containers)
- When segmenting on Unified Individual: if any linked individual matches criteria, unified individual joins
- Containers aggregate related attributes with: count, sum, average, max, min; up to 20 filters

### Segment Types
| Type | Refresh | Notes |
|---|---|---|
| Standard | 12 or 24 hours | Full capabilities |
| Rapid Publish | 1 or 4 hours | Only to MC Engagement Data Extension; max 20; 7-day engagement lookback |

### Segment Membership DMOs
Per published segment target, created automatically:
- `[Entity] Latest` DMO: current membership
- `[Entity] History` DMO: membership over time (includes Delta Type: added/removed/present)
- Use Query API, BYOL data share, or Tableau connector to access members externally

### Nested Segments
- Inner + outer segment must share same "segment on" entity
- Options: "Last Published Membership" (optimized) or "Segment Criteria" (copies criteria to outer)
- Only standard publish segments can be nested; rapid publish cannot

### Lookback Windows
- Engagement data: 2-year lookback (standard), 7 days (rapid)
- Use CIs to aggregate historical data beyond the lookback window

### Segmentation Best Practices
- Use **single container** for related attribute filters where possible (avoids UNION subqueries = fewer rows processed)
- For complex filter logic: define in a CI, then use CI as segment filter attribute
- Case-sensitive relationships: ensure values in object relationship paths have matching case
- Always include key qualifier fields when using CIs in segment containers
- Apply explicit lookback period filters on engagement data (default 2-year = all records processed)
- Stagger segment publishing times to avoid concurrent publish failures
- Use nested segments with "Last Published Membership" when membership changes infrequently
- Deactivate unused segments to avoid unnecessary credit consumption
- Choose shortest DMO relationship path when multiple paths available in a container

---

## 9. ACTIVATIONS AND DATA ACTIONS

### Activation Targets
File Storage (S3, GCS, Azure Blob, SFTP), Ad Platforms (Google Ads, DV360, LinkedIn, Meta, Amazon Ads), Salesforce MC Engagement, Salesforce B2C Commerce, Data Cloud DMO

### Activation Components
- **Activation Target**: credentials + connection to receiving platform
- **Activation Membership**: DMO containing segment members (e.g., Individual)
- **Contact Points**: Email, Phone, Mobile App, MAID, OTT, WhatsApp (optional for file storage)
- **Attributes**: Direct (one-to-one) or Related (one-to-many, delivered as JSON string)
- **Source Priority Order**: For unified individuals with multiple contact point values, determines which to use by ranked data source

### Contact Point & MC Engagement Rules
When MC Engagement is activation target and also contact point source:
1. Use highest Einstein Engagement Score
2. If undetermined: use lowest numerical Subscriber Id (oldest record)

### Consent Filtering
- Use Contact Point Consent DMO (Level 3) or Contact Point Subscription Consent DMO (Level 4)
- ⚠️ MC Connector maps Email Unsubscribe DLO to Email Engagement DMO but NOT to Engagement Channel Type Consent DMO — manually map those fields to enable opt-out filtering

### Activation Best Practices
- For MC Engagement activations: use full or incremental refresh; use incremental where possible
- Consider auto-suppression lists in MC Engagement as guardrail for unsubscribes
- For granular contact point control, use Individual (not Unified Individual) in activation — but be aware of potential duplicates; use query activity to filter
- Engagement data lookback in activation: 90 days (vs. 2-year segment lookback)

### Data Actions
- Trigger from streaming insights or record changes (CDC) on engagement DMOs
- Targets: Salesforce Platform Events (`DataObjectDataChgEvent`), MC Engagement, Webhook
- Webhook payload includes: `creationDateTime`, `count`, `schemas`, `events`; uses x-signature header for validation
- Optional: enrich with related object data; add event/action rules for conditional triggering
- Use cases: Flow to convert Lead→Contact on first purchase, journey injection on account activation, fulfillment trigger on order creation

---

## 10. REAL-TIME DATA

### Real-Time Capabilities
- Sub-second ingestion via SDK (web/mobile)
- Real-Time Data Graphs: pre-calculate and cache data from primary + related DMOs
- Real-Time Identity Resolution, Segmentation, Insights, and Actions
- Record Caching: up to 100M records; retention up to 180 days; configurable

### Requirements & Constraints
- Individual Id **required** in all SDK events for real-time graph routing; events without it route to standard pipeline
- Only insert/update supported in real-time; deletes must use standard pipeline
- Upserts don't remove old records immediately
- Billing: based on SSRT events + API calls

---

## 11. VECTOR SEARCH & RAG

### Unstructured Data Lake Objects (UDLOs)
- Store metadata for unstructured files (PDFs, transcripts, images, audio, contracts)
- Backend: Amazon S3, GCS, or Azure Blob Storage
- Auto-mapped to Unstructured Data Model Objects (UDMOs)

### Chunking Strategies (auto-selected by content type)
1. **Semantic-Based Passage Extraction**: uses HTML headings for topic boundaries
2. **Window-Based Passage Extraction**: uses block-level tags (div); less accurate
3. **Conversation-Based Chunking**: for transcripts; segments by speaker changes

Prepend fields (e.g., Title) to chunks to improve RAG prompt relevance.

### Vectorization Models
| Model | Input | Use Case |
|---|---|---|
| E5-Large V2 | English text | English semantic search |
| Multilingual E5-Large | 100+ language text | Cross-lingual retrieval |
| Whisper-Large-V3 | Audio | Speech-to-text for NLP |

### Retrieval-Augmented Generation (RAG) Flow
1. Prompt template invokes retriever in Data Cloud
2. Query vectorized → semantic search against index
3. Relevant context returned
4. Prompt augmented with context
5. LLM generates grounded response

### Retrievers
- Default retriever auto-created per search index (not customizable)
- Custom retrievers: configurable filters, field selection, result count limits (via Einstein Studio)
- Ensemble retrievers: aggregate multiple retrievers, re-rank by relevance
- Dynamic retrievers: runtime placeholders for personalized queries
- Citations: link responses to source content; enabled per retriever

---

## 12. PREDICTIVE AI MODELS (Einstein Studio)

### Model Types
- **Regression for Numbers**: predict numeric value (CLV, revenue, discount %)
- **Binary Outcomes**: predict one of two outcomes (churn/retain, win/lose, accept/decline)

### Algorithms
- Generalized Linear Model (GLM / Linear Regression)
- Gradient Boosted Machine (GBM)
- XGBoost

Einstein auto-recommends algorithm based on training data.

### Consumers (who/what uses model output)
Flows, Prediction Jobs, Batch Data Transforms, Agents, Prompt Templates, REST Applications, Apex (via auto-created invocable action)

### Consumption Patterns
- **On-Demand**: real-time via Flow/Apex/Predict API; ephemeral (not stored unless written)
- **Batch**: large volume at scheduled intervals; writes to DLO, Transform DMO, or ML Prediction DMO
- **Streaming**: on incoming event data; persisted in ML Prediction DMO; supports CDC-triggered flows

### BYOM (Bring Your Own Model)
Supported platforms: Amazon SageMaker, Google Cloud Vertex AI, Databricks
- Zero-copy: data stays in Salesforce; only input passed to external model API
- Configure: REST endpoint, auth, input/output schema

---

## 13. CONSENT

### Four Consent Levels
| Level | DMO | What It Covers |
|---|---|---|
| 1 | Individual | Broad opt-in/out flags (Do Not Market, Do Not Process, Do Not Track, Forget Me, Export Data) |
| 2 | Contact Point Type Consent | Which channels (email, phone) + preferred time/timezone + Data Use Purpose + Legal Basis |
| 3 | Contact Point Consent | Specific address-level consent (e.g., work email vs. personal email) |
| 4 | Communication Subscription / Consent | Content type consent (newsletter, announcements) by channel |

### Consent API
Supports: Processing restriction, Portability (data export), Right to be forgotten (PII deletion)

### Consent Considerations
- Data Cloud is **not a Consent Management Platform** — no cookie banners, preference centers, or automated retention/deletion
- Consent DMOs are for storage + use in segmentation/activation (consent filters) only
- Managing multiple consents per individual by contact point is **not currently supported**

---

## 14. SALESFORCE CRM INTEGRATION

### Integration Features
| Feature | Requires CRM Home Org |
|---|---|
| CRM Connector | No (also works standalone) |
| Data Cloud Triggered Flow | No |
| Related List Enrichments | Yes |
| Copy Field Enrichments | Yes |
| Reports and Dashboards | Yes |

### Data Cloud One (Multi-Org Topology)
- Home Org + Companion Orgs pattern
- Companion orgs: read-only access to shared Data Spaces; Query Editor is the only write-capable feature
- Metadata synchronized; data stays in home org
- Data residency/sovereignty must align with home org's geographic location
- Sandbox companion connections are NOT copied; must be re-established manually

---

## 15. CONSUMPTION & PRICING

### Three Components
1. **Data Storage**: terabytes stored in data model
2. **Data Services Credits**: compute usage per action
3. **Add-Ons**: Data Spaces, Advertising Audiences, Activations, Segmentation

### Key Credit Multipliers (per 1M rows unless noted)
| Service | Type | Credits |
|---|---|---|
| Batch Data Pipeline | Batch | 2,000 |
| Streaming Data Pipeline | Stream | 5,000 |
| Calculated Insights | Batch | 15 |
| Calculated Insights | Stream | 800 |
| Profile Unification | Batch | 100,000 |
| Data Actions | Stream | 800 |
| Real-Time Events | N/A | 70,000 (per 1M combined events+API+actions) |
| Data Queries | N/A | 2 |
| Real-time Profile API | N/A | 900 |
| Unstructured Data (Vector DB) | Batch | 60 per MB |
| Data Federation / Sharing | N/A | 70 per 1M rows |
| Inferences | N/A | 3,500 per 1M |

### Segment & Activation Credits
| Usage | Credits per 1M rows |
|---|---|
| Segment | 20 |
| Batch Activation | 10 |
| Streaming Activation | 1,600 |

### Consumption Best Practices
- Streaming ingestion costs ~2.5x more than batch — validate real-time requirement
- Profile Unification has highest multiplier (100,000/1M) — import only active profiles
- Batch ingestion: use **incremental** refresh over full refresh where possible
- Calculated Insights: merge dimensions into single multi-dimensional CI instead of separate CIs
- Segments: apply explicit engagement lookback filters; use single containers; use nested segments with Last Published Membership
- Activations: use incremental refresh; deactivate unused activations
- BYOL: only share objects needed; same AWS region = no credit cost for shared rows
- Monitor with Digital Wallet; purchase additional credits or investigate anomalies proactively

---

## 16. SANDBOXES & DEPLOYMENT

### Sandbox Types
Developer, Developer Pro, Partial Copy, Full Copy (not scratch orgs)
- Separate sandbox-specific SKUs for consumption (same multipliers as production)
- Monitor sandbox consumption from Digital Wallet in production org

### Deployment Options

**Change Sets**:
- Group components into a Data Kit per Data Space → add to outbound change set → upload and deploy
- Limitations: no rollback, no version control, no metadata diffing, manual component selection
- Data Cloud metadata has complex process-driven dependencies not handled automatically by package.xml

**Manifest-Based Deployment** (recommended, toward CI/CD):
```bash
# Create project
SFDX: Create Project with Manifest

# Retrieve metadata
sf project retrieve start --manifest package.xml

# Deploy to production
sf project deploy start --manifest package.xml
```
- Requires DevOps Data Kit with defined publishing sequence
- Authorization credentials are NOT replicated; re-authorize connectors after deployment

---

## 17. ARCHITECT METHODOLOGY

### Five-Stage Implementation Approach
1. **Discover**: Workshops with stakeholders → business challenges, requirements, use cases
2. **Audit**: Data dictionary per source (name, type, size, format, constraints, relationships)
3. **Design**: Map sources to standard DMOs; extend with custom only where needed
4. **Profile**: Identify profile data points; define match + reconciliation rules
5. **Document**: HLD (stakeholders) + LLD (technical team)

### User Story Framework
> "As a [role], I want [feature], so that [outcome]"

Start with a small number of high-value use cases; avoid scope creep.

### High-Level Design Document (HLD)
- Audience: business stakeholders, sponsors
- Contains: executive summary, business challenges, requirements, architecture diagram, use case definitions, system landscape, process flow diagrams

### Low-Level Design Document (LLD)
- Audience: data engineers, architects, developers
- Contains: data dictionary, ingestion connectors + refresh modes, DLO→DMO mapping, transform logic, data spaces, identity resolution rules, CI/streaming insight definitions, API integration details, segmentation + activation config, data actions, security/permission sets, provisioning decisions

### Process Flow Diagrams
- Preferred over UML component diagrams for stakeholder communication
- Show platforms, data flows, and component sequence per use case

---

## 18. DATA EGRESS

| Method | Use Case |
|---|---|
| JDBC Driver | BI/ETL tools via ANSI SQL queries |
| Tableau Connector | DLO/DMO/CIO querying; live or extract mode; supports data spaces |
| Data Graphs | JSON snapshot of related DMOs for primary Profile DMO; configurable refresh (hourly to monthly); up to 9 related objects, 3 relationship hops |
| Query API V1 | Synchronous freeform SQL; small extracts |
| Query API V2 | Paginated batches for large volume reads |
| Profile API | Retrieve profile data using search filters |
| Universal ID Lookup API | All records for a unified individual across sources |
| Calculated Insights API | Retrieve CI attributes/measures with dimension/measure filters |
| Data Graph API | Retrieve data graph data + metadata |
| MC Intelligence TotalConnect | Up to 50,000 rows/query, 1,000 queries/day |
| MC Intelligence Granular Data Center | Large-volume reads; uses Data Cloud Query API (billable) |

---

## 19. GLOSSARY OF KEY TERMS

- **CIO**: Calculated Insight Object — result of a CI run
- **CDLO/CDMO**: Chunk Data Lake/Model Object — used for vector search chunking
- **DLO**: Data Lake Object — physical Parquet store
- **DMO**: Data Model Object — virtual, non-materialized view
- **DSO**: Data Source Object — temporary raw staging
- **FQK**: Fully Qualified Key — source key + key qualifier, avoids conflicts
- **IDLO/IDMO**: Index Data Lake/Model Object — stores vector embeddings
- **UDLO/UDMO**: Unstructured Data Lake/Model Object — metadata for unstructured files
- **BYOL**: Bring Your Own Lake — zero-ETL federation with Snowflake/BigQuery/Databricks/Redshift
- **BYOM**: Bring Your Own Model — external ML model integration (SageMaker/Vertex AI/Databricks)
- **RAG**: Retrieval-Augmented Generation — grounds LLM prompts with indexed Data Cloud content
- **FQK**: Fully Qualified Key — source key + qualifier to prevent DMO key conflicts
- **Medallion Architecture**: Bronze (DSO) → Silver (DLO) → Gold (DMO) data layering pattern
