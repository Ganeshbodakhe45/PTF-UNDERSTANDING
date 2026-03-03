# PTF Position Massloading Module - Complete Project Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Module Architecture](#module-architecture)
3. [Folder Structure](#folder-structure)
4. [Connected Modules](#connected-modules)
5. [Key Components](#key-components)
6. [API Endpoints](#api-endpoints)
7. [Data Flow & Workflow](#data-flow--workflow)
8. [Configuration & Properties](#configuration--properties)
9. [Database Access Layer](#database-access-layer)
10. [Technologies & Dependencies](#technologies--dependencies)
11. [How to Run](#how-to-run)
12. [Important Notes for Beginners](#important-notes-for-beginners)

---

## Project Overview

### What is this project?

The **siti-ptf-mig-backend** is a large financial portfolio management system being modernized to a Spring Boot microservices architecture. The **ptf-position-massloading** module is specifically responsible for importing and processing financial position data in bulk through file uploads.

### What does "Position Massloading" do?

**In simple terms**: This module allows users to upload Excel or other file formats containing financial position data (stocks, bonds, derivatives, cash holdings, etc.) and validates, imports, and stores this data into the system.

### Key Features:
- **Excel File Upload**: Accept bulk position data from external sources
- **Validation**: Check for errors and inconsistencies in the data
- **Batch Processing**: Process large volumes of positions efficiently
- **Multiple Asset Classes**: Support for cash, securities, derivatives, and other financial instruments
- **Error Reporting**: Generate detailed error reports for rejected positions
- **Async Messaging**: Use event-driven architecture to process data asynchronously

---

## Module Architecture

### Clean Architecture Pattern

This project follows **Clean Architecture** principles with clear layer separation:

```
┌─────────────────────────────────────────────────────────┐
│                    API Layer (REST)                      │
│         PositionMassloadingController                    │
│  - Handles HTTP requests from frontend/clients           │
│  - Upload files, trigger jobs, retrieve results          │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                 Service Layer                           │
│  - PositionsIntegrationMassloadingService                │
│  - SitiPTFBatchStatusService                             │
│  - Job orchestration and business logic                  │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│         Infrastructure & Data Access Layer              │
│  - DAOs (Data Access Objects)                            │
│  - Job/Batch Processing                                  │
│  - Messaging & Event Handling                            │
│  - File I/O & External API Integration                   │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│            Database & External Systems                  │
│  - Oracle Database (via Spring Data JPA)                 │
│  - Message Queue (JMS)                                   │
│  - ECAR (Cash Account API)                               │
│  - Custody Securities API                                │
│  - Azure Data Lake (File Storage)                        │
└─────────────────────────────────────────────────────────┘
```

---

## Folder Structure

### ptf-position-massloading Module Structure

```
ptf-position-massloading/
├── pom.xml                          # Maven configuration file
├── README.adoc                      # AsciiDoc documentation
├── Jenkinsfile                      # CI/CD pipeline
├── SITI.jks                         # Java keystore (SSL certificates)
│
└── src/
    ├── main/
    │   ├── java/com/socgen/siti/ptf/api/ptfpositionmassloading/
    │   │   ├── PositionMassloadingApplication.java   # Spring Boot entry point
    │   │   ├── BatchConfig.java                       # Batch job configuration
    │   │   ├── DataLakeFeederConfig.java              # Azure Data Lake config
    │   │   ├── ECarSecuritiesRestTempConfig.java      # ECAR config
    │   │   │
    │   │   ├── api/                                   # REST API Layer
    │   │   │   └── dto/
    │   │   │       └── Characteristics.java            # API response DTO
    │   │   │
    │   │   ├── control/                               # REST Controllers
    │   │   │   ├── PositionMassloadingController.java  # Main API endpoint
    │   │   │   ├── PositionDuplicationController.java  # Duplication endpoint
    │   │   │   └── PTFJobRunnerController.java         # Batch job endpoint
    │   │   │
    │   │   ├── service/                               # Business Logic
    │   │   │   ├── PositionsIntegrationMassloadingService.java
    │   │   │   ├── SitiPTFBatchStatusService.java
    │   │   │   ├── FileReaderTools.java
    │   │   │   ├── ExecutionContextPopulator.java
    │   │   │   └── restclients/                       # External API clients
    │   │   │
    │   │   ├── dao/                                   # Data Access Objects
    │   │   │   ├── SitiPTFBatchStatusDao.java         # Interface
    │   │   │   ├── SitiPTFBatchStatusDaoImpl.java      # Implementation
    │   │   │   ├── PositionReportDM.java              # Position reports
    │   │   │   ├── RejectedPositionDM.java            # Rejected position data
    │   │   │   ├── FlowEventDM.java                   # Event data
    │   │   │   └── (Multiple DAO classes for different asset classes)
    │   │   │
    │   │   ├── job/                                   # Batch Job Framework
    │   │   │   ├── IJobRunner.java                    # Job runner interface
    │   │   │   ├── AbstractJobRunner.java             # Base job runner
    │   │   │   ├── JobResultDTO.java                  # Job result data
    │   │   │   ├── JobStatusDTO.java                  # Job status tracking
    │   │   │   ├── JobHelper.java                     # Utility methods
    │   │   │   └── ptf/                               # PTF-specific jobs
    │   │   │
    │   │   ├── business/                              # Business Layer
    │   │   │   ├── batch/
    │   │   │   │   ├── process/                       # Batch processors
    │   │   │   │   │   ├── DuplicateProcessTaskLet.java
    │   │   │   │   │   ├── DuplicationDecider.java
    │   │   │   │   │   ├── SitiJobSubscriberProcessor.java
    │   │   │   │   │   └── JobRunnerCheck.java
    │   │   │   │   ├── reader/                        # Data readers
    │   │   │   │   │   ├── JobJdbcPagingItemReader.java
    │   │   │   │   │   ├── JobReader.java
    │   │   │   │   │   └── JobRowMapper.java
    │   │   │   │   ├── writer/                        # Data writers
    │   │   │   │   │   └── JobWriter.java
    │   │   │   │   └── listener/                      # Batch listeners
    │   │   │   ├── applicative/
    │   │   │   ├── cashbalance/
    │   │   │   └── statementofholding/
    │   │   │
    │   │   ├── messaging/                             # Message Processing
    │   │   │   ├── MessagingConst.java                # Message constants
    │   │   │   ├── IMessagingController.java          # Message handler interface
    │   │   │   ├── MessagingControllerOS.java         # Implementation
    │   │   │   ├── DocumenttoClassBridge.java         # Message routing
    │   │   │   ├── SitiJobPublisher.java              # Job event publisher
    │   │   │   ├── JobRunnerStat.java                 # Job statistics
    │   │   │   │
    │   │   │   ├── data/                              # Message Data Models
    │   │   │   │   ├── MessageRoot.java               # Base message
    │   │   │   │   ├── MessageBody.java               # Message content
    │   │   │   │   ├── FxCashMessageRoot.java         # FX Cash message
    │   │   │   │   ├── SecuritySOHMessageRoot.java    # Security position message
    │   │   │   │   ├── CashStatementOfHoldingMessageRoot.java
    │   │   │   │   ├── ListedDerivativeMessageRoot.java
    │   │   │   │   ├── GenericOTCMessageRoot.java     # OTC derivatives
    │   │   │   │   ├── SwaptionMessageRoot.java       # Swaption messages
    │   │   │   │   ├── MessageLoader.java             # Message parsing
    │   │   │   │   └── (Many other message types)
    │   │   │   │
    │   │   │   ├── logs/                              # Message logging
    │   │   │   │   ├── IMsgLogStatementOfHoldingEvent.java
    │   │   │   │   ├── ReportLoggingProcess.java
    │   │   │   │   ├── MsgLogCashStatementOfHoldingEvent.java
    │   │   │   │   └── (Logging classes for each asset type)
    │   │   │   │
    │   │   │   └── subscriber/                        # Message Subscribers
    │   │   │       └── statementofholding/            # Position statement processors
    │   │   │           ├── IStatementOfHoldingProcess.java
    │   │   │           ├── AbstractStatementOfHoldingProcess.java
    │   │   │           ├── StatementOfHoldingJobSubscriber.java
    │   │   │           ├── StatementOfHoldingIntegrationBusinessProcess.java
    │   │   │           ├── cash/                      # Cash processors
    │   │   │           ├── cascade/                   # Cascade processors
    │   │   │           ├── security/                  # Security processors
    │   │   │           ├── otc/                       # OTC derivative processors
    │   │   │           ├── realestate/                # Real estate processors
    │   │   │           ├── listedderivative/          # Listed derivative processors
    │   │   │           └── subscribers/               # Job subscribers
    │   │   │
    │   │   ├── config/                                # Configuration Classes
    │   │   │   ├── ApiEcarConfiguration.java          # ECAR API setup
    │   │   │   ├── ClientEcarAPI.java                 # ECAR client
    │   │   │   ├── CustodySecuritiesConfiguration.java
    │   │   │   ├── CustodySecuritiesWebClientConfig.java
    │   │   │   ├── EcarHelperService.java
    │   │   │   ├── TradeManager.java                  # Trade processing
    │   │   │   ├── ParquetReader.java                 # Parquet file support
    │   │   │   └── (Many trade configuration classes)
    │   │   │
    │   │   ├── request/                               # API Request Models
    │   │   │   └── model/
    │   │   │       ├── Root.java
    │   │   │       ├── FiaResponseDto.java
    │   │   │       ├── RejectionDto.java
    │   │   │       └── (Other DTOs)
    │   │   │
    │   │   ├── modal/                                 # Modal/Data Models
    │   │   ├── model/                                 # Data Models
    │   │   ├── exception/                             # Custom Exceptions
    │   │   │   └── (Various exception classes)
    │   │   │
    │   │   └── StepExecutionLogger.java               # Batch step logging
    │   │
    │   └── resources/
    │       ├── application.yaml                        # Spring Boot configuration
    │       ├── bootstrap.yaml                          # Bootstrap configuration
    │       ├── logback-spring.xml                      # Logging configuration
    │       ├── ptf-position-massloading.sh             # Run script
    │       ├── com/                                    # Message schema definitions
    │       ├── distribution/                           # Distribution configs
    │       ├── static/                                 # Static resources
    │       └── (Certificate files: SITI.jks, ptf.jks)
    │
    └── test/
        └── java/
            └── (Unit and integration tests)
```

---

## Connected Modules

### Module Dependency Graph

```
ptf-position-massloading
├── siti_ptf_domain              (Business domain models)
├── siti_ptf_common              (Shared utilities)
├── siti_ptf_mapper              (Data mapping)
├── siti_ptf_ws_route            (Web service routing)
├── siti_ptf_report              (Reporting)
├── siti_ptf_report_domain       (Report domain models)
├── siti_ptf_route_api           (API routing)
└── External Libraries:
    ├── Spring Boot Framework
    ├── Spring Batch (batch processing)
    ├── Spring Data JPA (database)
    ├── Spring Security (authentication)
    ├── Apache POI (Excel handling)
    └── Apache Parquet (columnar data format)
```

### Key Connected Modules Explained:

#### 1. **siti_ptf_domain** (Business Domain Layer)
- **Location**: `siti_ptf_domain/`
- **Purpose**: Contains all domain models and business objects
- **What it provides**:
  - Position and statement of holding entities
  - Business validation rules
  - Event models for messaging
  - Exception definitions
- **Used by**: All other modules need domain objects

#### 2. **siti_ptf_common** (Shared Commons)
- **Location**: `siti_ptf_common/`
- **Purpose**: Shared utilities and enumerations used across all modules
- **What it provides**:
  - `RejectionCauseEnum` - Why positions are rejected
  - `SitiApplicationEnum` - Application identifiers
  - Common utility functions
  - Grid data structures

#### 3. **siti_ptf_mapper** (Object Mapping)
- **Location**: `siti_ptf_mapper/`
- **Purpose**: Converts objects between different layers
- **What it does**:
  - Maps domain models to DTOs (Data Transfer Objects)
  - Maps database entities to domain objects
  - Uses MapStruct for performance

#### 4. **siti_ptf_report** & **siti_ptf_report_domain** (Reporting)
- **Location**: `siti_ptf_report/` and `siti_ptf_report_domain/`
- **Purpose**: Generate reports from processed position data
- **Used by**: Position massloading for reporting processing results

#### 5. **siti_ptf_route_api** & **siti_ptf_ws_route** (API Routing)
- **Location**: `siti_ptf_route_api/` and `siti_ptf_ws_route/`
- **Purpose**: Route API requests and manage web services
- **Used by**: Request dispatcher in position massloading

---

## Key Components

### 1. REST Controllers

#### **PositionMassloadingController.java** (Main Controller)
```
Location: control/PositionMassloadingController.java
Lines: 803 lines
```

**What it does**:
- Exposes REST API endpoints for position importing
- Handles file uploads (Excel, CSV, Parquet)
- Validates uploaded files
- Triggers batch processing jobs
- Returns results and error reports

**Key Endpoints**:
- `POST /api/v1/positionintegration/upload` - Upload position file
- `GET /api/v1/positionintegration/status` - Check import status
- `POST /api/v1/positionintegration/validate` - Validate without importing

**Security**:
- Uses OAuth2 with scope: `api.ptf-position-integration.v1`
- CORS enabled for UI applications

---

#### **PositionDuplicationController.java**
- Handles position duplication requests
- Checks for duplicate positions in the system

---

#### **PTFJobRunnerController.java**
- Manages batch job lifecycle
- Start, stop, monitor jobs
- Check job execution status

---

### 2. Service Layer

#### **PositionsIntegrationMassloadingService.java**
**What it does**:
1. Reads Excel files containing position data
2. Extracts data into internal event objects
3. Validates positions through business rules
4. Creates rejection messages for invalid positions
5. Returns list of errors or confirmation of import

**Flow**:
```java
File (Excel)
    ↓
Read File → Extract Positions → Create SitiEvents
    ↓
Validate Events → Check Business Rules
    ↓
If Valid: Add to Queue
If Invalid: Create RejectionCauseEnum with error details
    ↓
Return PositionsIntegrationResult
```

---

#### **SitiPTFBatchStatusService.java**
**What it does**:
- Tracks batch job execution status
- Updates job status in database
- Manages job execution history
- Reports completion or failure

---

### 3. Data Access Objects (DAOs)

**Purpose**: Connect the business logic to the database

**Key DAOs**:

| DAO | Purpose |
|-----|---------|
| `SitiPTFBatchStatusDao` | Manage batch job records |
| `PositionReportDM` | Store position data |
| `RejectedPositionDM` | Track rejected positions |
| `FlowEventDM` | Store event flow information |
| `StatementOfHoldingIntegrationDM` | Store position statements |

---

### 4. Batch Processing

**Location**: `business/batch/`

The module uses **Spring Batch** for processing large volumes of positions efficiently.

#### Components:

1. **Readers** (`business/batch/reader/`)
   - `JobReader` - Reads data from database
   - `JobJdbcPagingItemReader` - Page-based JDBC reader
   - `JobRowMapper` - Maps database rows to objects

2. **Processors** (`business/batch/process/`)
   - `SitiJobSubscriberProcessor` - Main processing logic
   - `DuplicateProcessTaskLet` - Handles duplicate detection
   - `DuplicationDecider` - Decides on duplication logic
   - `JobRunnerCheck` - Validates job state

3. **Writers** (`business/batch/writer/`)
   - `JobWriter` - Writes processed data back to database

4. **Listeners** (`business/batch/listener/`)
   - Hook into batch lifecycle events
   - Log step execution
   - Handle errors

#### Batch Configuration:
From `BatchConfig.java`:
```yaml
Partition: 20          # Split into 20 partitions
Threads: 30            # Use 30 threads
Process Rows: 20000    # Process 20,000 rows per chunk
Chunk Size: 1          # Commit after each row (configurable)
```

---

### 5. Messaging System

**Location**: `messaging/`

The module uses **JMS (Java Message Service)** for asynchronous event processing.

#### Message Types (in `data/`):

1. **Position Messages**:
   - `SecuritySOHMessageRoot` - Security position
   - `CashStatementOfHoldingMessageRoot` - Cash position
   - `FxCashMessageRoot` - FX cash position
   - `ListedDerivativeMessageRoot` - Listed derivatives

2. **OTC Derivative Messages**:
   - `GenericOTCMessageRoot` - Generic OTC
   - `SwaptionMessageRoot` - Swaption
   - `CreditDerivativeMessageRoot` - Credit derivatives
   - `InterestRateSwapMessageRoot` - Interest rate swaps
   - `IndexEquitySwapMessageRoot` - Equity swaps

3. **Special Messages**:
   - `BorrLoandMessageRoot` - Borrowing loans
   - `FIACascadesMessageRoot` - FIA cascades
   - `RealEstateSOHMessageRoot` - Real estate holdings

#### Message Subscribers (in `subscriber/statementofholding/`):

Each message type has a processor:
- `CashStatementOfHoldingProcess` → Processes cash
- `SecurityStatementOfHoldingProcess` → Processes securities
- `OTCStatementOfHoldingProcess` → Processes derivatives
- `RealEstateStatementOfHoldingProcess` → Processes real estate

#### Message Flow:
```
Message Received (JMS)
    ↓
DocumenttoClassBridge → Route to correct processor
    ↓
StatementOfHoldingJobSubscriber → Extract data
    ↓
Validate (IStatementOfHoldingValidator)
    ↓
Process (IStatementOfHoldingProcess)
    ↓
Persist to Database
    ↓
Log Results (ReportLoggingProcess)
```

---

### 6. Configuration Layer

**Location**: `config/`

#### Trade Configuration:
```
TradeManager.java
├── TradeCash.java          - Cash trade configuration
├── TradeFX.java            - FX trade configuration
├── TradeIRSW.java          - Interest rate swap config
├── TradeEQSWAP.java        - Equity swap config
├── TradeListedDerivative.java - Listed derivatives
├── TradeSwaption.java      - Swaption config
├── TradeEQOPT.java         - Equity option config
└── TradeCDX.java           - Credit index config
```

Each trade configuration knows:
- How to parse the data
- How to validate it
- How to store it
- External system mappings

#### ECAR Integration:
```
ApiEcarConfiguration.java
├── ClientEcarAPI.java      - REST client for ECAR
├── EcarHelperService.java  - Utility methods
└── (ECAR = External Cash Account Repository)
```

---

## API Endpoints

### Main API Base Path
```
/api/v1/positionintegration
```

### Authentication
All endpoints require:
- OAuth2 token with scope: `api.ptf-position-integration.v1`

### Endpoints Overview

#### 1. **File Upload & Validation**

**POST** `/api/v1/positionintegration/upload`
- Upload Excel/CSV file with position data
- Validates file format and content
- Triggers batch import job
- Returns job ID and initial validation results

**Request**: MultipartFile (Excel or CSV)
**Response**:
```json
{
  "jobId": "12345",
  "status": "PROCESSING",
  "totalRecords": 1000,
  "validRecords": 950,
  "rejectedRecords": 50,
  "rejections": [
    {
      "rowNumber": 5,
      "cause": "INVALID_QUANTITY",
      "message": "Quantity must be positive"
    }
  ]
}
```

---

#### 2. **Check Import Status**

**GET** `/api/v1/positionintegration/status/{jobId}`
- Check current status of import job
- Returns completion percentage
- Lists any errors encountered

**Response**:
```json
{
  "jobId": "12345",
  "status": "COMPLETED",
  "progress": "100%",
  "startTime": "2026-03-03T10:00:00Z",
  "endTime": "2026-03-03T10:15:30Z",
  "totalProcessed": 1000,
  "successCount": 950,
  "failureCount": 50
}
```

---

#### 3. **Validate Before Import**

**POST** `/api/v1/positionintegration/validate`
- Validate data without actually importing
- Dry-run to catch errors early
- Useful for testing

---

#### 4. **Duplication Check**

**POST** `/api/v1/positionintegration/check-duplicates`
- Detect duplicate positions already in system
- Prevent accidental re-import

---

#### 5. **Job Management** (PTFJobRunnerController)

**POST** `/api/v1/positionintegration/job/start`
- Manually start a batch job

**POST** `/api/v1/positionintegration/job/stop`
- Stop a running job

**GET** `/api/v1/positionintegration/job/list`
- List all active jobs

---

## Data Flow & Workflow

### Complete Position Import Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: USER UPLOADS FILE                                       │
│ - User selects Excel file with positions                        │
│ - Sends to: POST /api/v1/positionintegration/upload             │
└────────────────┬────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────────┐
│ STEP 2: FILE VALIDATION (PositionMassloadingController)         │
│ - Check file format (Excel, CSV, Parquet)                       │
│ - Verify file size (max 20MB)                                   │
│ - Extract data from file                                        │
└────────────────┬────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────────┐
│ STEP 3: DATA EXTRACTION (PositionsIntegrationMassloadingService)│
│ - Read Excel sheets using Apache POI                            │
│ - Extract each position into SitiEvent objects                  │
│ - Map to internal data structures                               │
└────────────────┬────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────────┐
│ STEP 4: BUSINESS VALIDATION                                     │
│ - Check data types and formats                                  │
│ - Validate business rules (e.g., quantity > 0)                  │
│ - Check against master data (securities, currencies, etc.)      │
│ - Identify asset type (Cash, Security, Derivative, etc.)        │
│                                                                  │
│ For each position:                                              │
│ ├─ If VALID → Add to success list                              │
│ └─ If INVALID → Create RejectionCauseEnum entry                │
└────────────────┬────────────────────────────────────────────────┘
                 │
│  ┌─────────────┴──────────────────┐
│  │ Split: Valid vs Invalid        │
│  └────────┬────────────────────────┘
│
┌──────────┴─────────────────────────────────────────────────────┐
│ STEP 5A: PROCESS VALID POSITIONS (ASYNC - MESSAGE QUEUE)       │
│                                                                 │
│ For each valid position:                                       │
│ ├─ Create message (FxCashMessageRoot, SecurityMessageRoot,    │
│ │   OTCMessageRoot, etc.)                                      │
│ ├─ Put on JMS message queue                                    │
│ └─ Async processors consume and handle                         │
│                                                                 │
│ Each processor:                                                │
│ ├─ Validates business rules (StatementOfHoldingValidator)      │
│ ├─ Applies trade-specific logic (TradeManager)                 │
│ ├─ Enriches data (ECAR, Custody Securities APIs)               │
│ ├─ Persists to database (DAOs)                                 │
│ └─ Logs completion                                             │
└─────────────────────────────────────────────────────────────────┘
│
│ ┌─────────────────────────────────────────────────────────────┐
│ │ STEP 5B: HANDLE INVALID POSITIONS (BATCH JOB)               │
│ │                                                              │
│ │ Spring Batch processes rejections:                          │
│ │ ├─ Read rejected positions                                  │
│ │ ├─ Generate detailed error reports                          │
│ │ ├─ Store in RejectedPositionDM                              │
│ │ └─ Make available for download                              │
│ └─────────────────────────────────────────────────────────────┘
│
├─────────────────────────────────────────────────────────────────┐
│ STEP 6: DUPLICATION CHECK (OPTIONAL)                            │
│ - Check if positions already exist in system                   │
│ - Mark duplicates for resolution                               │
└────────────────┬────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────────┐
│ STEP 7: RETURN RESULTS TO USER                                  │
│ - Job completed status                                         │
│ - Import summary (X successful, Y rejected)                    │
│ - Link to error report download                                │
│ - Persisted positions available for querying                   │
└─────────────────────────────────────────────────────────────────┘
```

---

### Position Processing by Asset Class

#### Cash Positions
```
Cash File → CashStatementOfHoldingMessageRoot → CashMessageProcessor
    → Validate currency, amount, account
    → Store in cash tables
```

#### Security Positions
```
Security File → SecuritySOHMessageRoot → SecurityProcessor
    → Look up security master data
    → Validate quantity, price
    → Call Custody Securities API for enrichment
    → Store security holdings
```

#### OTC Derivatives
```
Derivative File → GenericOTCMessageRoot → OTCProcessor
    → Determine derivative type (Swaption, CDS, etc.)
    → Route to specific processor (SwaptionProcessor, CDXProcessor)
    → Validate pricing, notional amounts
    → Store OTC data
```

#### Listed Derivatives
```
Futures/Options → ListedDerivativeMessageRoot → ListedDerivativeProcessor
    → Validate contract specifications
    → Check exchange info
    → Store listed derivative data
```

---

## Configuration & Properties

### Main Configuration File
**Location**: `src/main/resources/application.yaml`

```yaml
# Server Configuration
server:
  port: 8086

# File Upload Limits
spring:
  servlet:
    multipart:
      max-file-size: 20MB          # Maximum file size for upload
      max-request-size: 20MB       # Maximum request size

# Batch Processing Configuration
batch:
  jdbc:
    lob:
      max-length: 100000          # Max LOB size for batch metadata
  job:
    enabled: false                # Don't auto-start jobs
  datasource:
    dbcp2:
      connection-timeout: 20000    # Wait 20 seconds for DB connection
      minimum-idle: 9              # Keep 9 idle connections
      maximum-pool-size: 70        # Max 70 connections
      idle-timeout: 10000          # Close idle after 10 seconds
      max-lifetime: 1000           # Refresh connection after 1 second

# Custom Job Configuration
job:
  partition: 20                   # Divide work into 20 partitions
  threads: 30                     # Use 30 threads
  process-rows: 20000            # Process 20K rows per batch
  chunk-size-paralell: 10        # Parallel chunk size
  chunk-size: 1                  # Write chunk size

# External API Configuration
api:
  properties:
    custody-securities:
      base-url: https://custody-securities-api.fr.socgen/api/v1
      characteristicsFieldsFilteringToken: WE44dIdi3OmWY-tnzPtrcg==
```

### OAuth2 Configuration
```yaml
spring:
  security:
    oauth2:
      client:
        # ECAR Cash Account API
        registration:
          ecar_cash_account:
            provider: sgconnect
            client-id: f6a7665f-0ead-4a7d-9e69-10c715c7ca40
            scope:
              - api.sgss-institutional-cash-account.v1
              - api.sgss-institutional-cash-account.cash-accounts.read
        
        # ECAR Securities Account API
        ecar_security_account:
          provider: sgconnect
          client-id: f6a7665f-0ead-4a7d-9e69-10c715c7ca40
          scope:
            - api.sgss-institutional-securities-account.v1
            - api.sgss-institutional-securities-account.securities-accounts.read
        
        # OAuth2 Provider
        provider:
          sgconnect:
            issuer-uri: https://sgconnect-hom.fr.world.socgen:443/sgconnect/oauth2
```

### CORS Configuration
```
Allowed Origins:
- https://ptf-mig-hom4.fr.world.socgen
- https://ptf-mig-hom3.fr.world.socgen
- https://ptf-mig-hom2.fr.world.socgen
- https://ptf-mig-asb1.fr.world.socgen
- https://ptf-mig-prod.fr.world.socgen

Allowed Methods: GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
Allowed Headers: *
```

---

## Database Access Layer

### Core DAOs

#### SitiPTFBatchStatusDao
**Purpose**: Manage batch execution records

**Methods**:
- `saveBatchStatus()` - Save job execution status
- `getBatchStatus(jobId)` - Retrieve job status
- `updateBatchStatus()` - Update running job

#### PositionReportDM
**Purpose**: Persist position data

**Methods**:
- `savePosition()` - Insert position record
- `getPosition(positionId)` - Retrieve position
- `getAllPositions()` - List all positions

#### RejectedPositionDM
**Purpose**: Track positions that failed validation

**Methods**:
- `saveRejection()` - Log rejection reason
- `getRejectedPositions(jobId)` - Get all rejections for a job
- `getRejectionReport()` - Generate error report

#### StatementOfHoldingIntegrationDM
**Purpose**: Store position statement data

**Methods**:
- `saveStatement()` - Save position statement
- `getStatement(statementId)` - Retrieve statement

---

## Technologies & Dependencies

### Core Technologies

| Technology | Version | Purpose |
|------------|---------|---------|
| Java | 8+ | Programming language |
| Spring Boot | 2.x | Application framework |
| Spring Batch | 4.x | Batch processing |
| Spring Data JPA | 2.x | Database ORM |
| Spring Security | 5.x | Authentication |
| Maven | 3.6+ | Build tool |

### Key Libraries

| Library | Purpose |
|---------|---------|
| Apache POI | Read/write Excel files |
| Apache Parquet | Columnar data format (for big data) |
| Dozer | Object mapping (legacy) |
| MapStruct | Object mapping (modern) |
| SLF4j + Logback | Logging |
| Lombok | Reduce boilerplate |
| Jackson | JSON processing |
| Apache Commons | Common utilities |

### External Integrations

| System | Purpose |
|--------|---------|
| Oracle Database | Primary data store |
| JMS (ActiveMQ/IBM MQ) | Message queue |
| ECAR API | Cash account enrichment |
| Custody Securities API | Security master data |
| Azure Data Lake | File storage/archive |

---

## How to Run

### Prerequisites
1. **Java 8+** installed
2. **Maven 3.6+** installed
3. **Database**: Oracle DB configured
4. **Message Queue**: JMS provider (ActiveMQ, IBM MQ, etc.)
5. **OAuth2 Provider**: Access to sgconnect service

### Build Steps

```bash
# 1. Navigate to project root
cd C:\HOMEWARE\PTF\Project\Aparnas Backup\siti-ptf-mig-backend

# 2. Build all modules (recommended first time)
mvn clean install

# Or build just position-massloading module
cd ptf-position-massloading
mvn clean install

# 3. Run the application
mvn spring-boot:run

# Application starts on http://localhost:8086
```

### Configuration Files to Update

Before running, update these files:

**config/env_dev.properties** - Database connection details
```properties
db.url=jdbc:oracle:thin:@localhost:1521:XE
db.user=ptf_user
db.password=yourpassword
```

**src/main/resources/application.yaml** - Application settings
```yaml
server.port: 8086
spring.datasource.url: jdbc:oracle:thin:@localhost:1521:XE
```

### Docker/Containerization
If containerized, the Dockerfile would:
```dockerfile
FROM openjdk:11-jre-slim
COPY ptf-position-massloading-6.0.185-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
EXPOSE 8086
```

---

## Important Notes for Beginners

### 1. Module Dependencies Flow
Remember: **Lower layers don't know about higher layers**
```
API Layer → Service Layer → DAO Layer → Database
   ↓           ↓              ↓
(REST)    (Business)    (Persistence)

Dependencies always point DOWN, never UP!
```

### 2. Key Concepts

#### **Statement of Holding (SOH)**
- Represents what positions a fund/portfolio holds
- Includes quantity, price, valuation
- Can be for any asset class (cash, securities, derivatives)

#### **Position vs Statement**
- **Position**: Individual holding (e.g., 1000 shares of Apple)
- **Statement**: Summary of all holdings for a given date

#### **Rejection Cause Enum**
- Represents why a position couldn't be imported
- Examples:
  - `INVALID_QUANTITY` - Quantity must be > 0
  - `INVALID_PRICE` - Price can't be negative
  - `SECURITY_NOT_FOUND` - Security doesn't exist in master data
  - `INVALID_CURRENCY` - Currency not recognized

### 3. Async Processing Pattern

The module uses **two paths**:

**Synchronous (Immediate)**:
- File upload validation
- Basic error checking
- Returns to user immediately

**Asynchronous (Background)**:
- Message queue processing
- Database persistence
- Report generation
- User polls for completion status

### 4. Error Handling

When positions are rejected:
1. Error captured in `RejectionDto` with:
   - Row number in source file
   - Rejection cause (enum)
   - Error message (human-readable)
2. Stored in `RejectedPositionDM` table
3. Available for download in error report
4. User can fix and re-upload

### 5. File Format Support

Currently supported:
- **Excel** (.xls, .xlsx) - Via Apache POI
- **CSV** - Standard format
- **Parquet** - Big data columnar format
- **XML** - For structured financial data

### 6. Performance Tuning

From `application.yaml`:
```yaml
job.partition: 20        # More partitions = faster but more resource usage
job.threads: 30          # Increase for parallel processing
job.process-rows: 20000  # Larger chunks = better throughput, slower feedback
```

Adjust based on:
- Available memory
- Database capability
- File size
- Acceptable latency

### 7. Security Layers

**API Level**:
- OAuth2 authentication
- Scope-based authorization: `api.ptf-position-integration.v1`

**Database Level**:
- User context isolation
- Entity auditing
- Change tracking

**Message Level**:
- Message signing
- Encryption for sensitive data

### 8. Testing Strategy

```
Unit Tests (Fast)
├─ Service layer tests
├─ DAO layer tests
└─ Business logic tests

Integration Tests (Medium Speed)
├─ API endpoint tests
├─ Database integration
└─ Message queue tests

End-to-End Tests (Slow)
├─ Full import workflow
├─ File to database
└─ Error reporting
```

### 9. Logging

**Logging Levels**:
- `DEBUG` - Detailed variable values, method entry/exit
- `INFO` - Major milestones (file received, job started, completed)
- `WARN` - Recoverable errors
- `ERROR` - Critical failures

**Log Location**: Check `logback-spring.xml` for output directories

### 10. Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| File too large | Exceeds 20MB limit | Split into smaller files |
| Out of memory | Too many threads/partitions | Reduce `job.threads` in config |
| Database connection timeout | DB not responding | Check DB connectivity |
| OAuth2 errors | Token expired/invalid | Refresh authentication |
| Duplicate positions | Reimporting same data | Use duplication check first |

---

## Architecture Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                        Frontend / Client                        │
│              (Web UI, Batch Scripts, Other Services)            │
└──────────────────────────┬─────────────────────────────────────┘
                           │ HTTP/REST
                           ▼
┌────────────────────────────────────────────────────────────────┐
│                    API Layer (Port 8086)                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ PositionMassloadingController                            │  │
│  │  ├─ POST /upload         (File upload endpoint)          │  │
│  │  ├─ GET  /status         (Check job status)              │  │
│  │  ├─ POST /validate       (Dry-run validation)            │  │
│  │  └─ POST /check-duplicates (Duplication check)           │  │
│  └──────────────────────────────────────────────────────────┘  │
│  Spring Security (OAuth2) & CORS Filter                        │
└──────────────────────────┬─────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────┐
│                  Service Layer (Business Logic)                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ PositionsIntegrationMassloadingService                   │  │
│  │  └─ Extract, Validate, Transform Excel data             │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ SitiPTFBatchStatusService                                │  │
│  │  └─ Track job execution status                           │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ FileReaderTools                                          │  │
│  │  └─ Parse Excel, CSV, Parquet files                      │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────┬─────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
    ┌─────────┐      ┌──────────┐      ┌──────────────┐
    │ JMS Queue│      │Database  │      │External APIs │
    │           │      │  (DAOs)  │      │(ECAR, Custody)│
    └─────────┘      └──────────┘      └──────────────┘
        │                  │                  │
        ▼                  ▼                  ▼
    ┌─────────────────────────────────────────────────────────┐
    │            Infrastructure Layer                         │
    │  ┌─────────────┐  ┌────────────────┐  ┌─────────────┐  │
    │  │ Messaging   │  │ Batch Job      │  │ Config      │  │
    │  │ Subscriber  │  │ Framework      │  │ & Trade     │  │
    │  │ (Async)     │  │ (Spring Batch) │  │ Processing  │  │
    │  └─────────────┘  └────────────────┘  └─────────────┘  │
    └─────────────────────────────────────────────────────────┘
        │                  │                  │
        ▼                  ▼                  ▼
    ┌────────────────────────────────────────────────────────┐
    │                 Data Persistence Layer                 │
    │  ┌────────────┐  ┌──────────────┐  ┌───────────────┐  │
    │  │ Oracle DB  │  │ Azure Lake   │  │ Message Queue │  │
    │  │ (Positions)│  │ (Archive)    │  │ (Event Store) │  │
    │  └────────────┘  └──────────────┘  └───────────────┘  │
    └────────────────────────────────────────────────────────┘
```

---

## Data Model Overview

### Position Data Structure

```
Position
├── Identifier
│   ├── Security ID / Instrument ID
│   ├── Account Number
│   ├── Portfolio ID
│   └── Date
├── Attributes
│   ├── Quantity
│   ├── Unit Price
│   ├── Total Valuation (Qty × Price)
│   ├── Currency
│   └── Cost Basis
├── Classification
│   ├── Asset Class (Cash, Security, Derivative)
│   ├── Security Type
│   ├── Product Type
│   └── Trade Status
└── Risk Metrics
    ├── Delta (for derivatives)
    ├── Gamma
    ├── Vega
    └── Theta
```

### Message Structure

```
Message
├── Message Header (Technical)
│   ├── MessageId
│   ├── Timestamp
│   ├── Source System
│   ├── Document Type
│   └── Correlation ID
├── Functional Context
│   ├── Business Date
│   ├── Application Date
│   ├── User Context
│   └── Entity ID
└── Message Body (Position Data)
    ├── Position-specific details
    ├── Quantity and pricing
    ├── Account information
    └── Trade details
```

---

## Deployment Considerations

### Environment Variables Needed
```
MONITORING_ENABLED=true
SGMON_REALM=ptf-prod
ZIPKIN_CLIENTID=your-client-id
ZIPKIN_SECRET=your-client-secret
DB_URL=jdbc:oracle:thin:@prod-db:1521:XE
DB_USER=ptf_user
DB_PASSWORD=encrypted-password
```

### Port Requirements
- **8086** - Main API port
- **9090** - Actuator/Admin (health checks)

### Database Requirements
- **Connection Pool**: Min 9, Max 70 connections
- **LOB Storage**: Support for large objects (100KB+ binary data)
- **Tables**: Position, RejectedPosition, BatchStatus, etc.

### Message Queue Requirements
- **Queue Names**: Typically match message types
- **Throughput**: Should handle 20,000 messages/batch
- **Persistence**: Messages should be persisted for reliability

---

## Glossary

| Term | Definition |
|------|-----------|
| **SOH** | Statement of Holding - Summary of positions |
| **OTC** | Over-The-Counter - Non-exchange-traded derivatives |
| **ECAR** | External Cash Account Repository - Cash system API |
| **DAO** | Data Access Object - Database interface |
| **DTO** | Data Transfer Object - API data structure |
| **JMS** | Java Messaging Service - Message queue |
| **Batch** | Large-scale automated data processing |
| **Spring Batch** | Framework for batch processing |
| **Chunk** | Unit of work in batch processing (read-process-write) |
| **Partition** | Division of work across threads |
| **Tasklet** | Single task in batch step |
| **Processor** | Transforms input items to output items |
| **Rejection** | Position that failed validation |

---

## Summary

The **ptf-position-massloading** module is a sophisticated position import system that:

1. **Accepts** bulk position data from users via REST API
2. **Validates** data against business rules and master data
3. **Processes** valid positions asynchronously using Spring Batch and JMS
4. **Enriches** data by calling external APIs (ECAR, Custody Securities)
5. **Persists** positions to database using DAOs
6. **Tracks** rejected positions and provides error reports
7. **Reports** completion status back to users

It's designed for **high throughput**, **reliability**, and **ease of integration** with other PTF modules, all while maintaining **clean architecture** principles and **security best practices**.

The module serves as a critical data ingestion point for the entire portfolio management system, handling multiple asset classes and complex validation rules with an elegant async processing pattern.

---

**Document Created**: March 3, 2026  
**Version**: 1.0  
**Module Version**: 6.0.185-SNAPSHOT

