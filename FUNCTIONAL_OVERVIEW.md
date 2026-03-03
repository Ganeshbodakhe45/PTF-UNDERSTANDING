# SITI PTF Modernization Backend - Functional Overview

**Project Name:** siti-ptf-mig-backend  
**Version:** 6.0.185-SNAPSHOT  
**Organization:** Société Générale (SITI - Societe Informatique SITI)  
**Technology Stack:** Spring Boot, Java, Oracle Database  

---

## 1. Overall Purpose of the Project

### What is this system?

This is a **Portfolio Management System (PTF)** modernization project that has been migrated from a legacy system to a modern Spring Boot-based architecture. PTF stands for "Portfolio" and this system manages financial portfolios, positions, and related data for Société Générale's investment operations.

### What problem does it solve?

The project modernizes an older portfolio management system to:
- Provide REST APIs for portfolio and position management instead of legacy interfaces
- Enable concurrent handling of multiple portfolio operations (search, creation, loading, reporting)
- Support secure data access with OAuth2 authentication
- Offer real-time visibility into portfolio data, holdings, and certifications
- Handle bulk position uploads and mass loading operations
- Generate comprehensive reports on portfolio positions and certifications

### Who uses it?

The system serves:
- **Investment Managers:** Who need to query and manage portfolios
- **Risk Teams:** Who need real-time position data and certifications
- **Operations Teams:** Who perform bulk position uploads and data maintenance
- **Compliance/Reporting Teams:** Who generate and export portfolio reports
- **Internal Web Applications:** A web UI (PTF-MIG-GUI) that calls these APIs

---

## 2. Functional Flow

### High-Level User Workflows

#### **Workflow 1: Portfolio Search & Consultation**
```
User Login (OAuth2)
    ↓
Access Search/Consult API
    ↓
Query Portfolio by ID/Criteria
    ↓
Receive Portfolio Details (positions, holdings, certifications)
    ↓
Export/View Data
```

#### **Workflow 2: Portfolio Group Management**
```
User Login
    ↓
Access Portfolio Group API
    ↓
Create/Update/Query Portfolio Groups
    ↓
Manage Pool of Related Portfolios
    ↓
Track Group-Level Metrics
```

#### **Workflow 3: Position Mass Loading**
```
User Login
    ↓
Upload Position File/Feed
    ↓
System Validates Positions
    ↓
Process Batch Job (async)
    ↓
Update Database with New Positions
    ↓
Generate Status Report
```

#### **Workflow 4: Data Extraction & Integration**
```
System Scheduler/Manual Trigger
    ↓
Extract Data from Source Systems (BDR, SIR, SIA)
    ↓
Transform & Normalize Data
    ↓
Load into PTF Database
    ↓
Notify Systems of Completion
```

#### **Workflow 5: Report Generation & Certification**
```
User Request Integration Report
    ↓
System Processes Report
    ↓
Validate Inventory Data
    ↓
Certify Data Completeness
    ↓
Export Report (PDF/Excel)
```

### System Inputs & Outputs

**Inputs:**
- User queries and API requests (REST calls)
- Position data files (bulk uploads)
- Portfolio configuration changes
- Message feeds from external systems (BDR, SIR, SIA)
- Report generation requests

**Outputs:**
- Portfolio data (JSON/REST responses)
- Reports (PDF, Excel formats)
- Position processing status
- Integration certifications
- Audit logs and system events

---

## 3. Functional Role of Each Module/Component

### **Core Foundation Modules**

#### **`siti_ptf_common`**
- **Purpose:** Shared utilities and common definitions
- **Functional Role:** Provides cross-cutting concerns like user context, request context, enumerations, and constants used across all modules
- **What it does:** Defines SIR constants, user permissions, application request/user context for identifying who is accessing the system

#### **`siti_ptf_domain`**
- **Purpose:** Core business domain objects
- **Functional Role:** Represents the heart of the system - the business entities for Portfolio, Position, Holdings, Certifications, etc.
- **What it does:** Defines domain models that represent real-world portfolio concepts (what is a Portfolio, what data does it hold, what are the rules)

#### **`siti_ptf_common`**
- **Purpose:** Common utilities and base classes
- **Functional Role:** Base classes, interfaces, and utility functions for deep cloning, common DTO structures

### **Integration & Mapping Modules**

#### **`siti_ptf_mapper`**
- **Purpose:** Object-to-object mapping
- **Functional Role:** Converts between domain objects and API DTOs using the Dozer mapping framework
- **What it does:** Transforms internal portfolio objects into JSON-friendly data transfer objects for API responses and vice versa

#### **`siti_ptf_ws_client`**
- **Purpose:** Web service client library
- **Functional Role:** Encapsulates calls to external web services (BDR, SIR, SIA systems)
- **What it does:** Provides client code to fetch data from upstream financial systems

#### **`siti_ptf_ws_route`**
- **Purpose:** Web service routing and orchestration
- **Functional Role:** Routes data between components and external systems
- **What it does:** Uses mapping and client libraries to coordinate data flow between domain objects and external service calls

#### **`siti_ptf_route_api`**
- **Purpose:** API routing layer
- **Functional Role:** Defines routes and orchestration patterns for API interactions

### **Reporting & Analytics Modules**

#### **`siti_ptf_report_domain`**
- **Purpose:** Domain models specific to reporting
- **Functional Role:** Defines report-specific entities and data structures
- **What it does:** Represents report concepts (certifications, inventories, validations)

#### **`siti_ptf_report`**
- **Purpose:** Report generation and formatting
- **Functional Role:** Generates reports in multiple formats (PDF, Excel, etc.)
- **What it does:** Uses JasperReports library to create formatted portfolio reports with charts and tables

#### **`siti_ptf_batch`**
- **Purpose:** Batch processing framework
- **Functional Role:** Executes long-running jobs asynchronously
- **What it does:** Processes bulk data loads, scheduled extractions, and other background operations

### **REST API Services (Public Interfaces)**

#### **`ptf-proxy-api-service`**
- **Purpose:** API gateway/proxy service
- **Functional Role:** Acts as the main entry point for web UI and external clients
- **What it does:** Provides OAuth2 token handling, user validation, and proxies requests to backend services
- **Key Features:** Decryption services, user authentication validation, secure communication bridge

#### **`ptf-search-consult`**
- **Purpose:** Portfolio search and viewing API
- **Functional Role:** Allows users to query and view portfolio data
- **What it does:** Provides endpoints to search portfolios by criteria, retrieve position details, view holdings and certifications
- **Data Sources:** Queries Oracle database directly via DAO layer
- **Exports:** Can export data to Excel

#### **`ptf-portfolio-group`**
- **Purpose:** Portfolio grouping and pooling API
- **Functional Role:** Manages collections of related portfolios
- **What it does:** Create, update, delete, and query portfolio groups; manage criteria for group membership
- **Use Case:** Allows risk managers to organize portfolios into logical groups for analysis

#### **`ptf-position-massloading`**
- **Purpose:** Bulk position upload and processing
- **Functional Role:** Handles massive position data ingestion from feeds
- **What it does:** 
  - Accepts position feeds (Statement of Holding messages from various instruments: cash, derivatives, securities)
  - Validates positions against business rules
  - Processes asynchronously using Spring Batch
  - Updates position database with new holdings
  - Monitors job status and generates reports
- **Supported Asset Types:** Cash, Listed Derivatives, OTC Derivatives (FX Options, Index Equity Swaps, etc.)

#### **`ptf-integration-report`**
- **Purpose:** Data integration and certification reporting
- **Functional Role:** Validates that position data from all sources is complete and consistent
- **What it does:** 
  - Compares positions from different data sources
  - Certifies data completeness (inventory certification)
  - Generates integration reports
  - Can run standalone or as part of larger processes

#### **`ptf-configuration`**
- **Purpose:** System configuration management
- **Functional Role:** Manages portfolio configuration parameters
- **What it does:** Provides endpoints to manage configuration settings that control system behavior

#### **`ptf-extraction`**
- **Purpose:** Data extraction from source systems
- **Functional Role:** Pulls data from upstream financial systems
- **What it does:** 
  - Extracts positions from BDR (Business Data Repository), SIR (System Information Repository), SIA systems
  - Transforms extracted data into PTF format
  - Loads data into database
  - Generates extraction reports and validations

#### **`safe-url-service`**
- **Purpose:** Secure URL encryption/decryption
- **Functional Role:** Protects sensitive URLs in the system
- **What it does:** Encrypts URLs for secure parameter passing, decrypts and validates them on use
- **Use Case:** Prevents tampering with URLs that contain sensitive portfolio or user data

#### **`ptf-message-consumer`**
- **Purpose:** Asynchronous message processing
- **Functional Role:** Consumes messages from message queues
- **What it does:** Processes position update messages asynchronously, handles Statement of Holding (SOH) feed processing
- **Integration Pattern:** Event-driven architecture for real-time updates

---

## 4. Functional Features List

### **Portfolio Management Features**

| Feature | Description | Dependencies |
|---------|-------------|--------------|
| **Search Portfolios** | Find portfolios by ID, name, manager, or other criteria | ptf-search-consult, Oracle DB |
| **View Portfolio Details** | Display all positions, holdings, and certifications for a portfolio | ptf-search-consult |
| **Group Portfolios** | Create and manage logical groupings of related portfolios | ptf-portfolio-group |
| **Manage Group Pools** | Define criteria-based pools within portfolio groups | ptf-portfolio-group |
| **Filter & Sort** | Advanced filtering and sorting of portfolio lists | ptf-search-consult |
| **Export Data** | Export portfolio data to Excel or PDF formats | ptf-report, ptf-search-consult |

### **Position Management Features**

| Feature | Description | Dependencies |
|---------|-------------|--------------|
| **Upload Positions (Bulk)** | Load thousands of positions at once via feeds or files | ptf-position-massloading |
| **Validate Positions** | Check positions against business rules before loading | ptf-position-massloading |
| **Support Multiple Asset Types** | Handle cash, securities, derivatives (OTC and listed) | ptf-position-massloading |
| **Real-time Position Updates** | Consume position update messages asynchronously | ptf-message-consumer |
| **Duplication Detection** | Identify and handle duplicate positions | ptf-position-massloading |
| **Position History Tracking** | Track position changes over time | ptf-search-consult |

### **Data Integration Features**

| Feature | Description | Dependencies |
|---------|-------------|--------------|
| **Extract from BDR** | Pull data from Business Data Repository | ptf-extraction, ws-client |
| **Extract from SIR** | Pull data from System Information Repository | ptf-extraction, ws-client |
| **Extract from SIA** | Pull data from SIA system | ptf-extraction, ws-client |
| **Data Transformation** | Convert external data formats to PTF standard | ptf-extraction, siti_ptf_mapper |
| **Load Data** | Insert transformed data into database | ptf-extraction |
| **Validate Integration** | Ensure all sources are in sync | ptf-integration-report |

### **Reporting & Certification Features**

| Feature | Description | Dependencies |
|---------|-------------|--------------|
| **Generate Position Reports** | Create detailed position reports with filters | ptf-report |
| **Inventory Certification** | Certify that all positions from all sources are present | ptf-integration-report |
| **Data Completeness Validation** | Verify no missing positions or data gaps | ptf-integration-report |
| **Consolidation Reports** | Aggregate positions across portfolios or groups | ptf-integration-report, ptf-report |
| **Export Reports** | Generate PDF or Excel reports | siti_ptf_report |
| **Scheduled Reports** | Run reports on a schedule | siti_ptf_batch |

### **Security & Configuration Features**

| Feature | Description | Dependencies |
|---------|-------------|--------------|
| **OAuth2 Authentication** | Secure user login and token-based access | ptf-proxy-api-service |
| **Authorization/Permissions** | Control which users can access which portfolios | All API services |
| **Encrypted URLs** | Prevent URL tampering with sensitive data | safe-url-service |
| **Configuration Management** | Manage system-wide settings and parameters | ptf-configuration |
| **SSL/TLS Support** | Secure communication over HTTPS | All services |

### **System Operation Features**

| Feature | Description | Dependencies |
|---------|-------------|--------------|
| **Batch Job Execution** | Run long-running operations asynchronously | siti_ptf_batch |
| **Job Monitoring** | Track status of batch jobs | ptf-position-massloading, siti_ptf_batch |
| **Error Handling** | Graceful handling of failures with detailed error messages | All services |
| **Audit Logging** | Log all user actions for compliance | All services |
| **Health Checks** | Monitor system health and availability | All services (Spring Actuator) |

---

## 5. Functional Data Understanding

### **Core Portfolio Data**

**Portfolios**
- What it represents: A container for financial positions held by a manager or entity
- What it stores: Portfolio metadata (ID, name, manager, status), and references to positions
- Used by: Search/Consult, Portfolio Group, Position queries

**Positions**
- What it represents: A single holding of an asset (stock, bond, derivative, cash, etc.)
- What it stores: Position details (asset type, quantity, market value, cost, counterparty, etc.)
- Used by: All position-related operations

**Holdings/Statements of Holding (SOH)**
- What it represents: Historical records of position ownership at a point in time
- What it stores: Position snapshots with timestamps
- Used by: Position tracking, history views, reconciliation

**Certifications**
- What it represents: Data quality certifications confirming positions are complete and accurate
- What it stores: Certification metadata (certified date, source, validation status)
- Used by: Integration reports, compliance reporting

**Portfolio Groups/Pools**
- What it represents: Logical collections of related portfolios
- What it stores: Group definitions, membership criteria, pool parameters
- Used by: Portfolio organization, group-level reporting

### **Configuration Data**

**Generic Parameters**
- What it represents: System configuration settings
- What it stores: Key-value pairs for tuning system behavior
- Used by: Extraction processes, report generation, API configuration

**Source System Mappings**
- What it represents: Rules for converting external data formats
- What it stores: Field mappings, transformation logic
- Used by: Extraction and data transformation

### **Report Data**

**Integration Reports**
- What it represents: Reconciliation between data from different sources
- What it stores: Comparison results, discrepancies, certifications
- Used by: Compliance, quality assurance

**Position Reports**
- What it represents: Formatted, human-readable position summaries
- What it stores: Formatted position data, charts, totals
- Used by: Portfolio managers, risk teams

---

## 6. Configurations from a Functional Perspective

### **Environment Configuration** (`config/env_dev.properties`)

| Parameter | What it Controls | Functional Impact |
|-----------|------------------|-------------------|
| `ws.bdr.host.socgen` | BDR connection address | Where the system fetches position data from BDR |
| `ws.bdr.port.socgen` | BDR connection port | Communication channel to BDR |
| `ws.sir.host.socgen` | SIR connection address | Where the system fetches SIR reference data |
| `ptf.ds.url` | Portfolio database location | Which database stores portfolio data |
| `ptf.ds.user` / `ptf.ds.password` | Database credentials | Who can access the database |
| `siti.ptf.api.host` | API server hostname | Where users connect to call APIs |
| `siti.ptf.api.integrationreport.port` | Integration report service port | Which port serves integration reports |
| `sgconnect.authorize.uri` | OAuth2 authorization server | How users authenticate to the system |
| `ptf.ssl.keystore.*` | SSL certificate configuration | Secure HTTPS communication settings |

**Functional Impact of Changing Configs:**
- Changing database URL redirects system to different portfolio data
- Changing OAuth URI prevents users from logging in
- Changing service ports makes APIs unreachable
- Changing BDR/SIR addresses breaks data extraction

---

## 7. Functional Integration Points

### **External Systems Integration**

#### **BDR (Business Data Repository)**
- **What it is:** Legacy source system for position data
- **What flows in:** Position queries, extract requests
- **What flows out:** Position records, reference data
- **Functional purpose:** Primary source of truth for financial positions
- **How accessed:** Via ptf-extraction module using web service clients

#### **SIR (System Information Repository)**
- **What it is:** System containing entity and instrument reference data
- **What flows in:** Master data queries
- **What flows out:** Instrument definitions, counterparty data, corporate actions
- **Functional purpose:** Enriches position data with descriptive information
- **How accessed:** Via ptf-extraction module

#### **SIA (SIA System)**
- **What it is:** Alternative or complementary position data source
- **What flows in:** Position queries
- **What flows out:** Position data, holdings
- **Functional purpose:** Validates data consistency across sources
- **How accessed:** Via ptf-extraction module

#### **PTF Web UI (ptf-mig-gui)**
- **What it is:** Browser-based user interface for portfolio management
- **What flows in:** User interactions (clicks, searches, uploads)
- **What flows out:** REST API calls to backend services
- **Functional purpose:** Provides user-friendly access to portfolio system
- **How accessed:** Via ptf-proxy-api-service (gateway service)

#### **Message Queue / Event Bus**
- **What it is:** Asynchronous messaging system for event-driven processing
- **What flows in:** Position update messages, feed notifications
- **What flows out:** Processed position updates, job status notifications
- **Functional purpose:** Enables real-time, decoupled processing of data changes
- **How accessed:** Via ptf-message-consumer module

#### **OAuth2 / SGConnect (Authentication Service)**
- **What it is:** Centralized authentication provider
- **What flows in:** Login credentials
- **What flows out:** Access tokens, user authorization info
- **Functional purpose:** Secures all API access to portfolio system
- **How accessed:** Via ptf-proxy-api-service

---

## 8. Functional Gaps & Observations

### **Missing Documentation**
1. ❌ Many module README files contain template placeholders ("SHORT INTRODUCTION", "INFORMATION NEEDED")
2. ❌ No detailed API documentation visible for the actual endpoints and payloads
3. ❌ No clear user guide or operation manual
4. ❌ No disaster recovery or backup procedures documented

### **Ambiguous/Unclear Functional Areas**
1. 🤔 **Relationship between ptf-integration-report and ptf-search-consult:** Both appear to provide portfolio data, but their exact functional division is unclear
2. 🤔 **Portfolio vs. Portfolio Group distinction:** Not clear when/why a user would use groups vs. individual portfolio queries
3. 🤔 **Data extraction frequency:** No documentation on how often data is pulled from BDR/SIR/SIA (daily? hourly? real-time?)
4. 🤔 **Message processing flow:** unclear how ptf-message-consumer relates to ptf-position-massloading (which handles bulk uploads)

### **Functionally Risky Areas**
1. ⚠️ **Dual Position Update Paths:** Both position-massloading (batch) and message-consumer (async) can update positions - risk of conflicts or duplicates
2. ⚠️ **Multiple Data Sources (BDR, SIR, SIA):** Data reconciliation could be complex; unclear how conflicts are resolved
3. ⚠️ **No documented data retention policy:** How long are position histories kept? What happens to old certifications?
4. ⚠️ **Extraction job scheduling:** No visible documentation on what triggers extractions and how frequently they run
5. ⚠️ **Error handling in position loading:** What happens if a batch job fails partway through? Is rollback supported?

### **Potential User Impact Issues**
1. 📌 **API Gateway Complexity:** Users cannot call services directly; must go through ptf-proxy-api-service, which could be a single point of failure
2. 📌 **Performance with Large Portfolios:** No documentation on maximum portfolio size or position count supported
3. 📌 **Report Generation Latency:** Unclear how long report generation takes (could be minutes/hours for large datasets)
4. 📌 **Authentication Token Expiry:** No visible user-facing guidance on token refresh or re-authentication
5. 📌 **Search/Consult Query Limits:** No mention of result set limits or pagination strategies

### **Architecture Concerns**
1. 🔴 **Circular Dependencies Risk:** Multiple modules depend on siti_ptf_domain and siti_ptf_mapper - any changes could break multiple services
2. 🔴 **Tight Coupling to Oracle DB:** Direct JDBC/Tomcat JDBC Pool usage means switching databases would be difficult
3. 🔴 **No visible transaction coordination:** How are distributed transactions handled across extraction and loading?
4. 🔴 **Test coverage unclear:** Many modules have test exclusions in pom.xml (sonar.exclusions), suggesting incomplete coverage

### **Missing Functional Features** (Not Evident)
1. 🚫 **Position Deletion:** No clear API for removing obsolete positions
2. 🚫 **Bulk Position Correction:** How are incorrect position data corrected once loaded?
3. 🚫 **Audit Trail Display:** No clear way for users to see who changed what and when
4. 🚫 **Position Alerting:** No mention of alerts for anomalies (e.g., positions exceeding thresholds)
5. 🚫 **Rollback/Reversal:** No documented way to undo batch operations

---

## 9. System Architecture Summary

### **Layered Architecture**

```
┌─────────────────────────────────────────────────────────────────┐
│  Presentation Layer (Web UI)                                     │
│  ptf-mig-gui (Browser-based interface)                          │
└────────────────────────┬────────────────────────────────────────┘
                         │ (HTTPS/REST calls)
┌────────────────────────▼────────────────────────────────────────┐
│  API Gateway / Security Layer                                    │
│  ptf-proxy-api-service (Authentication, OAuth2 validation)      │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
┌────────▼──────┐ ┌─────▼──────┐ ┌─────▼───────────┐
│  REST APIs    │ │  Batch     │ │  Message        │
│               │ │  Processing│ │  Consumer       │
│  - Search     │ │            │ │                 │
│  - Consult    │ │ - Position │ │ - Process SOH   │
│  - Portfolio  │ │   Loading  │ │   Messages      │
│  - Portfolio  │ │ - Reports  │ │ - Update        │
│    Groups     │ │ - Extraction
│  - Config     │ │            │ │  Positions      │
│  - Safe URL   │ │            │ │                 │
└────────┬──────┘ └─────┬──────┘ └────────┬────────┘
         │               │               │
         └───────────────┼───────────────┘
                         │
         ┌───────────────▼───────────────┐
         │  Service Layer                │
         │  - Business logic             │
         │  - Orchestration              │
         │  - Mapping & transformation   │
         └───────────────┬───────────────┘
                         │
         ┌───────────────▼───────────────┐
         │  Data Access Layer (DAO)      │
         │  - Database queries           │
         │  - External system calls      │
         └───────────────┬───────────────┘
                         │
         ┌───────────────▼───────────────┐
         │  Data Sources                 │
         │  - Oracle Portfolio DB        │
         │  - BDR (external API)         │
         │  - SIR (external API)         │
         │  - SIA (external API)         │
         │  - Message Queues             │
         └───────────────────────────────┘
```

---

## 10. Key Business Processes

### **Daily Operations Flow**

1. **Morning:** Systems pull data from BDR/SIR/SIA (via ptf-extraction)
2. **Throughout Day:** 
   - Users query portfolios (ptf-search-consult)
   - Real-time position updates arrive via messages (ptf-message-consumer)
3. **Afternoon:** Batch position loads are processed (ptf-position-massloading)
4. **End of Day:** 
   - Integration validation reports generated (ptf-integration-report)
   - Data certifications completed
   - Portfolio managers export reports (ptf-report)

### **Exception Handling Flow**

- Position validation fails → Error logged, batch job paused, operations team notified
- Data mismatch between sources → Integration report flags discrepancy, investigation triggered
- User authentication fails → Redirected to login via sgconnect OAuth2

---

## 11. Deployment Units

The system is deployed as multiple independent Spring Boot microservices:

| Service | Deployment | Purpose |
|---------|-----------|---------|
| `ptf-proxy-api-service` | Docker container | API Gateway |
| `ptf-search-consult` | Docker container | Portfolio search API |
| `ptf-portfolio-group` | Docker container | Portfolio grouping API |
| `ptf-position-massloading` | Docker container | Position loading API |
| `ptf-integration-report` | Docker container | Report generation API |
| `ptf-extraction` | Docker container | Data extraction service |
| `ptf-configuration` | Docker container | Configuration API |
| `ptf-message-consumer` | Docker container | Async message processor |
| `safe-url-service` | Docker container | URL encryption service |

Each service can be scaled independently and has its own database connections.

---

## Summary

The **SITI PTF Modernization Backend** is a comprehensive portfolio management platform that provides:

✅ **User-Facing Functionality:** Portfolio search, viewing, grouping, and reporting  
✅ **Operations Functionality:** Bulk position loading, data extraction, integration validation  
✅ **Security:** OAuth2-based authentication, encrypted URLs, secure communication  
✅ **Integration:** Connections to legacy systems (BDR, SIR, SIA) and message queues  
✅ **Scalability:** Microservices architecture allowing independent scaling  

⚠️ **Main Challenges:** Multiple data sources requiring reconciliation, complex batch processing, need for better documentation and error handling clarity.


