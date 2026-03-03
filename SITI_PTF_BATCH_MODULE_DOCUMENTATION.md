# SITI PTF Batch Module - Complete Documentation

**Last Updated:** March 2026  
**Module Version:** 6.0.185-SNAPSHOT  
**Module Name:** `siti_ptf_batch`  
**Type:** Batch Processing Module (Java)

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Module Purpose](#module-purpose)
3. [Architecture & Design](#architecture--design)
4. [Folder Structure](#folder-structure)
5. [Core Components](#core-components)
6. [Configuration System](#configuration-system)
7. [Batch Types & Examples](#batch-types--examples)
8. [Error Handling](#error-handling)
9. [Dependencies & Technologies](#dependencies--technologies)
10. [How to Run](#how-to-run)
11. [Testing](#testing)
12. [Key Observations & Notes](#key-observations--notes)

---

## Project Overview

### What is This Module?

The `siti_ptf_batch` module is a **batch processing framework** for the **SITI PTF** system. It's a Java-based command-line application that:

- Executes batch jobs remotely by calling REST APIs
- Manages authentication with OAuth2/SG-Connect
- Handles various PTF operations (data extraction, position updates, archiving, etc.)
- Provides error handling and recovery mechanisms
- Supports multiple batch configurations

### Context

This is part of a **multi-module Maven project** called `siti-ptf-mig-backend`. It represents the **Infrastructure (Infra) Layer** in the Clean Architecture pattern, as it's the entry point for batch operations.

---

## Module Purpose

The batch module serves as a **gateway** between external schedulers (like job runners) and the PTF microservices. Instead of making direct API calls from external systems, this batch:

1. **Loads configuration files** that define what API to call
2. **Authenticates** with the PTF backend using OAuth2 credentials
3. **Makes REST API calls** to PTF microservices
4. **Handles errors** and provides return codes for external systems
5. **Supports different batch types**:
   - **SIMPLE_CALL**: Direct API call and return the response
   - **PARAM_UNIT**: Batched processing with independent transactions

---

## Architecture & Design

### High-Level Workflow

```
┌─────────────────────────────────────────────────────────────┐
│ External Scheduler (e.g., Job Control System)                │
│ Calls: java -jar siti_ptf_batch.jar <config_file>           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │   APIBatch.main()          │
        │   (Entry Point)            │
        └────────┬───────────────────┘
                 │
    ┌────────────▼─────────────────┐
    │ Load Configuration Files      │
    │ (batch.properties + specific) │
    └────────┬─────────────────────┘
             │
    ┌────────▼──────────────────┐
    │ Create Batch Context      │
    │ (User, Entity, Client ID) │
    └────────┬──────────────────┘
             │
    ┌────────▼─────────────────────────────┐
    │ Get OAuth2 Access Token from SG-Connect
    │ (SgConnectAccessTokenServiceImpl)      │
    └────────┬─────────────────────────────┘
             │
    ┌────────▼───────────────────────┐
    │ Build REST Request              │
    │ (Headers + Batch Input DTO)     │
    └────────┬───────────────────────┘
             │
    ┌────────▼──────────────────────────┐
    │ Call PTF Microservice API         │
    │ (Configuration/Extraction/etc.)   │
    └────────┬──────────────────────────┘
             │
    ┌────────▼──────────────────────┐
    │ Evaluate Response              │
    │ (Check Status Code & Body)     │
    └────────┬──────────────────────┘
             │
    ┌────────▼──────────────────────┐
    │ Handle Errors (if any)        │
    │ (IBatchHandler implementation) │
    └────────┬──────────────────────┘
             │
    ┌────────▼──────────────────────┐
    │ Return Exit Code              │
    │ 0 = Success, 1 = Warning, 2 = Error
    └──────────────────────────────┘
```

### Design Patterns Used

1. **Template Method Pattern**: `IBatchHandler` interface defines error handling behavior
2. **Strategy Pattern**: Different batch error handlers for different batch types
3. **Dependency Injection**: Spring framework manages components
4. **Factory Pattern**: `BatchPropertyNameEnum` maps batch names to configuration categories
5. **Singleton Pattern**: `SgConnectAccessTokenServiceImpl` maintains a single OAuth2 token

### Key Classes Relationships

```
APIBatch
  ├─ Uses: BatchContext (holds user & auth info)
  ├─ Uses: IBatchHandler (error handling strategy)
  │  └─ Implemented by: DefaultBatchErrorHandler
  │  └─ Implemented by: Various PTFBatch*ErrorHandler classes
  ├─ Uses: SgConnectAccessTokenServiceImpl (OAuth2)
  ├─ Uses: BatchPropertyNameEnum (config mapping)
  └─ Uses: RestTemplate (HTTP calls)

JobRunnerClient
  └─ Similar structure to APIBatch but specific to JobRunner

PtfFlowIntegrationClient
  └─ Similar structure to APIBatch but specific to Flow Integration
```

---

## Folder Structure

```
siti_ptf_batch/
│
├── pom.xml                          # Maven configuration & dependencies
├── .classpath, .project, .settings/ # Eclipse IDE configuration
├── README.md                        # Project readme (if exists)
│
├── src/
│   ├── main/
│   │   ├── java/com/socgen/siti/ptf/batch/
│   │   │   ├── client/
│   │   │   │   ├── APIBatch.java                         # Main entry point
│   │   │   │   ├── BatchContext.java                     # Context/state holder
│   │   │   │   ├── BatchPropertyNameEnum.java            # Config mapping
│   │   │   │   ├── DefaultBatchErrorHandler.java         # Default error handler
│   │   │   │   ├── IBatchHandler.java                    # Error handler interface
│   │   │   │   ├── SgConnectAccessTokenServiceImpl.java   # OAuth2 token service
│   │   │   │   ├── jobrunner/
│   │   │   │   │   └── JobRunnerClient.java              # JobRunner-specific batch
│   │   │   │   └── flowintegration/
│   │   │   │       └── PtfFlowIntegrationClient.java     # Flow Integration batch
│   │   │   └── error/
│   │   │       ├── PTFBatchBrinExtractionErrorHandler.java
│   │   │       ├── PTFBatchCashCollateralDuplicatedPositionErrorHandler.java
│   │   │       ├── PTFBatchCashDepositDuplicatedPositionErrorHandler.java
│   │   │       ├── PTFBatchCashExternalDuplicatedPositionErrorHandler.java
│   │   │       ├── PTFBatchDamasExtractionErrorHandler.java
│   │   │       ├── PTFBatchDateApplicativeErrorHandler.java
│   │   │       ├── PTFBatchDuplicatedPositionBatchTestErrorHandler.java
│   │   │       ├── PTFBatchFluxValorisateurErrorHandler.java
│   │   │       ├── PTFBatchGlobalTestConnectionErrorHandler.java
│   │   │       ├── PTFBatchJobRunnerErrorHandler.java
│   │   │       ├── PTFBatchListedDerivativeDuplicatedPositionErrorHandler.java
│   │   │       ├── PTFBatchOTCDuplicatedPositionErrorHandler.java
│   │   │       ├── PTFBatchPurgeArchivageErrorHandler.java
│   │   │       ├── PTFBatchSecuritiesLendingDuplicatedPositionErrorHandler.java
│   │   │       └── PTFBatchXwiseSimcorpFlowErrorHandler.java
│   │   │
│   │   └── resources/
│   │       ├── batch.properties                          # Global batch configuration
│   │       ├── log4j.properties                          # Logging configuration
│   │       ├── jobrunner.properties                      # JobRunner config
│   │       ├── PTFBatchBrinExtraction.properties         # Data extraction config
│   │       ├── PTFBatchDamasExtraction.properties        # Damas extraction config
│   │       ├── PTFBatchDateApplicative.properties        # Date update config
│   │       ├── PTFBatchFluxValorisateur.properties       # Valuation flow config
│   │       ├── PTFBatchCashCollateralDuplicatedPosition.properties
│   │       ├── PTFBatchCashDepositDuplicatedPosition.properties
│   │       ├── PTFBatchCashExternalDuplicatedPosition.properties
│   │       ├── PTFBatchListedDerivativeDuplicatedPosition.properties
│   │       ├── PTFBatchOTCDuplicatedPosition.properties
│   │       ├── PTFBatchSecuritiesLendingDuplicatedPosition.properties
│   │       ├── PTFBatchDuplicatedPositionBatchTest.properties
│   │       ├── PTFBatchPurgeArchivage.properties
│   │       ├── PTFBatchConfigGlobalConnectionTest.properties
│   │       ├── PTFBatchFluxValorisateur.properties
│   │       ├── PTFBatchXWiseSimcorp.properties
│   │       ├── PTFBatchXWiseSimcorpDataLake.properties
│   │       ├── PTFFlowIntegration.properties
│   │       ├── ftr_conf.txt                              # FTR configuration
│   │       ├── *.sh files (deployment scripts)
│   │       ├── *.cfg files (flow configuration)
│   │       ├── sqlScripts/                               # SQL maintenance scripts
│   │       │   ├── 01_archivageptf_init.sql
│   │       │   ├── 02_archivageptf_tablesref.sql
│   │       │   ├── ...other SQL scripts...
│   │       │   └── vm_bdr_controle_ptf.sql
│   │       │
│   │       └── Shell Scripts (for deployment):
│   │           ├── ptfBrinExtraction.sh
│   │           ├── ptfDamasExtraction.sh
│   │           ├── ptfJobRunner.sh
│   │           ├── ptfFluxValorisateurPTF.sh
│   │           ├── ptfCashCollateralDuplicatedPosition.sh
│   │           ├── ptfGlobalTestConnection.sh
│   │           └── ...more batch scripts...
│   │
│   └── test/
│       └── java/com/socgen/siti/ptf/batch/
│           └── client/
│               ├── APIBatchTest.java                     # Unit tests
│               ├── flowintegration/
│               └── jobrunner/
│
├── src/assembly/
│   └── assembly.xml                 # Maven assembly descriptor for packaging
│
├── target/                          # Build output (compiled classes, JARs)
│
└── .gitignore                       # Git ignore rules

```

---

## Core Components

### 1. **APIBatch.java** (Main Entry Point)

**Location**: `src/main/java/com/socgen/siti/ptf/batch/client/APIBatch.java`

**Purpose**: The main executable class that serves as the entry point for batch execution.

**Key Methods**:

| Method | Purpose |
|--------|---------|
| `main(String[] args)` | Entry point - receives config file path and parameters |
| `execute(List<String> params)` | Orchestrates the entire batch execution flow |
| `readPropertiesFile(String fileName)` | Loads configuration from property files |
| `getProperty(String key)` | Retrieves a configuration property |
| `startBatchProcessing()` | Determines batch type and starts processing |
| `buildAndCallAPI()` | Constructs HTTP request and calls the PTF API |
| `evaluateAPIResponse(ResponseEntity<String> response)` | Validates the API response |
| `handleGatewayIssue(int returnValue)` | Handles gateway timeouts with polling mechanism |
| `getSitiApiBatchJobStatus(String jobIdentifier)` | Polls for job status during timeout recovery |

**Key Features**:

- **Parameter Loading**: Accepts batch configuration file path as first argument
- **OAuth2 Authentication**: Gets access token from SG-Connect before calling APIs
- **Token Management**: Caches OAuth2 token and regenerates if expired
- **Error Handling**: Maps return codes to batch handler strategies
- **Gateway Timeout Recovery**: If API doesn't respond, polls for job status every 5 minutes
- **Unique Job Identifier**: Generates unique ID for each batch run (entity + user + URI + random + timestamp)

**Example Usage**:

```bash
# Run BRIN extraction batch
java -jar siti_ptf_batch.jar C:\config\PTFBatchBrinExtraction.properties

# Run with additional parameters (used by some batch types)
java -jar siti_ptf_batch.jar C:\config\PTFBatchFluxValorisateur.properties param1 param2
```

**Return Codes**:

- `0`: Success
- `1`: Non-critical error (batch can continue)
- `2`: Critical error (batch must stop)

---

### 2. **BatchContext.java** (State Holder)

**Location**: `src/main/java/com/socgen/siti/ptf/batch/client/BatchContext.java`

**Purpose**: Encapsulates all context information needed for a batch run (extends `ApplicationUserContext`).

**Properties**:

| Property | Type | Purpose |
|----------|------|---------|
| `clientId` | String | OAuth2 client ID for authentication |
| `clientSecret` | String | OAuth2 client secret for authentication |
| `sgConnectAccessTokenURI` | String | SG-Connect URL for getting OAuth2 token |
| `ptfApiHost` | String | Hostname of PTF API (e.g., localhost, prod server) |
| `ptfConfigurationApiPort` | String | Port for configuration API (e.g., 8087) |
| `ptfExtractionApiPort` | String | Port for extraction API (e.g., 8088) |
| `ptfPositionIntegrationApiPort` | String | Port for position integration API (e.g., 8086) |
| `ptfJobRunnerPort` | String | Port for JobRunner (if used) |
| `jobUniqueIdentifier` | String | Unique ID for this job (used for status polling) |
| *Inherited from ApplicationUserContext* | | |
| `userId` | String | User ID running the batch |
| `userLogin` | String | User login/username |
| `entityFunctionalId` | String | Functional entity ID (e.g., SGPAR) |
| `entityId` | Long | Numeric entity ID (e.g., 6) |
| `locale` | Locale | Language/locale (en, fr) |

**Example**:

```java
BatchContext ctx = new BatchContext();
ctx.setClientId("205a38bf-42c0-4a80-954e-2aca41370225");
ctx.setClientSecret("2dn37l5ahgi0174k51cfb0g7hfi31");
ctx.setUserId("ACCBATCH");
ctx.setUserLogin("ACCBATCH");
ctx.setEntityId(6L);
```

---

### 3. **IBatchHandler.java** (Error Handling Strategy)

**Location**: `src/main/java/com/socgen/siti/ptf/batch/client/IBatchHandler.java`

**Purpose**: Interface that defines error handling and lifecycle callbacks for batch execution.

**Methods**:

| Method | Purpose |
|--------|---------|
| `onServiceException(Throwable e)` | Called when API/service call fails |
| `onParamException(Throwable e)` | Called when parameter parsing fails |
| `onTechnicalError(int code, Throwable e)` | Called for technical errors (config, args, etc.) |
| `onCompleteServiceProcess(int code)` | Called after service execution completes |
| `onCompleteParmsProcess(String[][] values)` | Called after parameter processing |
| `onCompleteWholeBatch(int code)` | Called when entire batch finishes |
| `setBatchParams(List<String> params)` | Sets the batch parameters |
| `getBatchParams()` | Retrieves the batch parameters |

**Return Values** (defined in `APIBatchConstants`):

- `0` = Success (RETURN_OK)
- `1` = Non-critical error
- `2` = Critical error (stop batch)
- `3` = Critical error (tasks batch)

---

### 4. **DefaultBatchErrorHandler.java** (Default Implementation)

**Location**: `src/main/java/com/socgen/siti/ptf/batch/client/DefaultBatchErrorHandler.java`

**Purpose**: Default implementation of `IBatchHandler` that provides basic error handling.

**Behavior**:

- Returns critical error code for all exceptions
- Logs errors
- Doesn't perform any special recovery

**Example Override**:

```java
public class PTFBatchBrinExtractionErrorHandler extends DefaultBatchErrorHandler {
  @Override
  public int onServiceException(Throwable e) {
    logger.error(e.getMessage(), e);
    // Can implement custom retry logic here
    return APIBatchConstants.NON_CRITICAL_ERROR;  // Allow batch to continue
  }
}
```

---

### 5. **SgConnectAccessTokenServiceImpl.java** (OAuth2 Token Service)

**Location**: `src/main/java/com/socgen/siti/ptf/batch/client/SgConnectAccessTokenServiceImpl.java`

**Purpose**: Manages OAuth2 token acquisition and caching for API authentication.

**Key Behavior**:

1. **Token Caching**: Keeps token in memory until it expires
2. **Automatic Refresh**: Regenerates token if it's null or expired
3. **Base64 Encoding**: Encodes client credentials for Basic Auth
4. **Grant Type**: Uses `client_credentials` grant type

**Process**:

```
getAccessToken() called
    │
    ├─ Token exists AND not expired?
    │   └─ YES → Return cached token
    │
    └─ NO → Call generateAccessToken()
              │
              ├─ Base64 encode client credentials
              ├─ Build URI with grant_type and scope
              ├─ POST to SG-Connect OAuth2 endpoint
              ├─ Receive OAuth2AccessToken
              └─ Cache it for reuse
```

**Example Response**:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "api.ptf-extraction.v1"
}
```

---

### 6. **BatchPropertyNameEnum.java** (Configuration Mapping)

**Location**: `src/main/java/com/socgen/siti/ptf/batch/client/BatchPropertyNameEnum.java`

**Purpose**: Maps batch property file names to batch types and API categories.

**Batch Types Supported**:

| Enum Value | Config File | API Category | Purpose |
|------------|------------|--------------|---------|
| `PTF_APPLICATIVE_DATE` | PTFBatchDateApplicative | Configuration | Update application date |
| `PTF_BRIN_EXTRACTION` | PTFBatchBrinExtraction | Extraction | Extract BRIN data |
| `PTF_DAMAS_EXTRACTION` | PTFBatchDamasExtraction | Extraction | Extract DAMAS data |
| `PTF_FLUX_VALORISATEUR` | PTFBatchFluxValorisateur | Extraction | Run valuation flow |
| `PTF_CASH_COLLATERAL_DUPLICATION_POSITION` | PTFBatchCashCollateralDuplicatedPosition | Position Integration | Duplicate cash collateral positions |
| `PTF_CASH_DEPOSIT_DUPLICATION_POSITION` | PTFBatchCashDepositDuplicatedPosition | Position Integration | Duplicate cash deposit positions |
| `PTF_OTC_DUPLICATION_POSITION` | PTFBatchOTCDuplicatedPosition | Position Integration | Duplicate OTC positions |
| `PTF_PURGE_ARCHIVAGE` | PTFBatchPurgeArchivage | Configuration | Archive old data |
| `PTF_JOB_RUNNER` | jobrunner | Position Integration | Control JobRunner service |
| `PTF_FLOW_INTEGRATION` | PTFFlowIntegration | Position Integration | Flow integration processing |

**Method**: `retrieveBatchApiCategory(String batchPropertyName)` returns the API port to use based on batch type.

---

### 7. **Error Handler Classes** (Batch-Specific)

**Location**: `src/main/java/com/socgen/siti/ptf/batch/error/`

**Purpose**: Each batch type can have its own error handling strategy.

**Pattern**: All extend `DefaultBatchErrorHandler` and override specific behaviors.

**Example - PTFBatchBrinExtractionErrorHandler**:

```java
public class PTFBatchBrinExtractionErrorHandler extends DefaultBatchErrorHandler {
  private static int mCounter = 0;  // Tracks successful executions
  
  @Override
  public int onServiceException(Throwable e) {
    logger.error(e.getMessage(), e);
    // Return non-critical error - allows batch to continue
    return APIBatchConstants.NON_CRITICAL_ERROR;
  }
  
  @Override
  public void onCompleteWholeBatch(int returnCode) {
    logger.info("PtF Batch Brin extraction Completed :" + IBatchLogCode.BRPR004);
    mCounter = 0;  // Reset counter
  }
}
```

---

### 8. **JobRunnerClient.java** (JobRunner-Specific Batch)

**Location**: `src/main/java/com/socgen/siti/ptf/batch/client/jobrunner/JobRunnerClient.java`

**Purpose**: Specialized batch client for controlling the JobRunner service.

**Key Differences from APIBatch**:

- Handles JobRunner-specific configuration
- Manages thread pool parameters (number of threads, polling interval, cache settings)
- Checks `jobRunnerRunning` flag in response JSON
- May require different port configuration

**Supported Parameters**:

- `jobrunner.thread.number`: Number of worker threads
- `jobrunner.thread.polling.interval`: How often to poll for updates (ms)
- `jobrunner.cache.min`: Minimum cache size
- `jobrunner.cache.max`: Maximum cache size
- `jobrunner.cache.refresh.interval`: Cache refresh rate (ms)

---

### 9. **PtfFlowIntegrationClient.java** (Flow Integration Batch)

**Location**: `src/main/java/com/socgen/siti/ptf/batch/client/flowintegration/PtfFlowIntegrationClient.java`

**Purpose**: Specialized batch client for flow integration operations.

**Characteristics**:

- Handles PTF flow integration batches
- Similar structure to APIBatch and JobRunnerClient
- Specific error handling via `PTFBatchFlowIntegrationErrorHandler`

---

## Configuration System

### Configuration File Structure

Each batch has TWO configuration files:

1. **Global Config**: `batch.properties` - Shared by all batches
2. **Batch-Specific Config**: `PTFBatch*.properties` - Specific to each batch type

### Global Configuration (`batch.properties`)

**Location**: `src/main/resources/batch.properties`

```properties
# User Context
batch.userId=ACCBATCH                                    # User ID
batch.userLogin=ACCBATCH                               # User Login
batch.entityFunctionalId=SGPAR                          # Functional entity
batch.entityId=6                                        # Entity numeric ID
batch.local=en                                          # Language (en/fr)

# OAuth2 Credentials
batch.clientId=205a38bf-42c0-4a80-954e-2aca41370225   # OAuth2 client
batch.clientSecret=2dn37l5ahgi0174k51cfb0g7hfi31      # OAuth2 secret

# API Endpoints
batch.ptfApiHost=localhost                              # Server hostname
batch.ptfConfigurationApiPort=8087                     # Config API port
batch.ptfExtractionApiPort=8088                        # Extraction API port
batch.ptfPositionIntegrationApiPort=8086               # Position API port

# OAuth2 Token Endpoint
batch.sgConnectAccessTokenURI=https://sgconnect-hom.fr.world.socgen/sgconnect/oauth2/access_token
```

**Usage**: These values are loaded for every batch run and provide the context.

---

### Batch-Specific Configuration Example

**File**: `PTFBatchBrinExtraction.properties`

```properties
# Batch metadata
NUMBER_OF_THREADS = 1                                   # Parallel threads
BATCH_TYPE = SIMPLE_CALL                                # Type: SIMPLE_CALL or PARAM_UNIT
BATCH_ID = 1                                            # Numeric ID

# API Details
API_SCOPE = api.ptf-extraction.v1                       # OAuth2 scope
REMAINING_API_URI = /api/v1/extractions/brin/processExtraction  # API endpoint

# Error Handling
BATCH_ERROR_HANDLER=com.socgen.siti.ptf.batch.error.PTFBatchBrinExtractionErrorHandler
```

### Batch Type Explanation

| Type | Meaning | Example |
|------|---------|---------|
| **SIMPLE_CALL** | Call API once, return response immediately | Update date, check connection |
| **PARAM_UNIT** | Process with multiple threads/transactions independently | Position duplication, data loading |

### Configuration Loading Flow

```
Program starts with: java -jar ... PTFBatchBrinExtraction.properties

1. Read batch.properties (global config)
   ├─ Load user context (userId, userLogin, entity)
   ├─ Load OAuth2 credentials
   └─ Load API endpoints

2. Read PTFBatchBrinExtraction.properties (batch-specific)
   ├─ Load BATCH_TYPE (SIMPLE_CALL or PARAM_UNIT)
   ├─ Load API_SCOPE
   ├─ Load REMAINING_API_URI
   └─ Load BATCH_ERROR_HANDLER class name

3. Create BatchContext with all loaded properties

4. Instantiate error handler class (if specified)

5. Execute batch using all configuration
```

---

## Batch Types & Examples

### Batch 1: BRIN Extraction

**Configuration**: `PTFBatchBrinExtraction.properties`

**API Called**: `POST /api/v1/extractions/brin/processExtraction`

**Port**: 8088 (Extraction API)

**Purpose**: Extract BRIN (Securities Identification Number) data

**Execution**:

```bash
java -jar siti_ptf_batch.jar PTFBatchBrinExtraction.properties
```

---

### Batch 2: Damas Extraction

**Configuration**: `PTFBatchDamasExtraction.properties`

**API Called**: `POST /api/v1/extractions/damas/processExtraction`

**Purpose**: Extract DAMAS (Data Management System) data

---

### Batch 3: Flux Valorisateur (Valuation Flow)

**Configuration**: `PTFBatchFluxValorisateur.properties`

**API Called**: `POST /api/v1/extractions/fas/processExtraction`

**Batch Type**: `PARAM_UNIT`

**Purpose**: Process valuation flows and FAS calculations

---

### Batch 4: Update Applicative Date

**Configuration**: `PTFBatchDateApplicative.properties`

**API Called**: `POST /api/v1/configurations/updateApplicationDate`

**Port**: 8087 (Configuration API)

**Purpose**: Set the current business date for the system

---

### Batch 5: Cash Collateral Position Duplication

**Configuration**: `PTFBatchCashCollateralDuplicatedPosition.properties`

**API Called**: `POST /api/v1/positionduplication/processCashCollateral`

**Port**: 8086 (Position Integration API)

**Status API**: `POST /api/v1/positionduplication/getSitiApiBatchStatus`

**Batch Type**: `PARAM_UNIT`

**Purpose**: Duplicate cash collateral position records

**Number of Threads**: 5 (parallel processing)

---

### Batch 6: Purge Archivage

**Configuration**: `PTFBatchPurgeArchivage.properties`

**API Called**: `POST /api/v1/configurations/processPtfPurgeArchivage`

**Purpose**: Archive and purge old data

**Features**: 
- Supports status polling via `REMAINING_API_STATUS_URI`
- Handles long-running operations

---

### Batch 7: JobRunner Control

**Configuration**: `jobrunner.properties`

**API Called**: `POST /api/v1/jobrunner/controljobrunner`

**Port**: 8086 (Position Integration API)

**Purpose**: Control and monitor the JobRunner service

**Specific Parameters**:

```properties
jobrunner.thread.number=20                   # 20 worker threads
jobrunner.thread.polling.interval=500        # Poll every 500ms
jobrunner.cache.min=20                       # Min cached objects
jobrunner.cache.max=200                      # Max cached objects
jobrunner.cache.refresh.interval=500         # Cache refresh every 500ms
```

---

### Batch 8: Flow Integration

**Configuration**: `PTFFlowIntegration.properties`

**API Called**: `POST /api/v1/jobrunner/process-flow-integration`

**Purpose**: Process flow integration operations

---

## Error Handling

### Error Handler Interface

Each batch defines an error handler implementing `IBatchHandler`:

```java
public interface IBatchHandler {
  int onServiceException(Throwable e);              // API call failed
  int onParamException(Throwable e);                // Parameter error
  int onTechnicalError(int code, Throwable e);      // Config/technical error
  void onCompleteServiceProcess(int code);          // After service call
  void onCompleteParmsProcess(String[][] values);   // After parameter parse
  void onCompleteWholeBatch(int code);              // Batch completion
  void setBatchParams(List<String> params);
  List<String> getBatchParams();
}
```

### Error Handler Configuration

Each batch specifies its error handler in properties:

```properties
BATCH_ERROR_HANDLER=com.socgen.siti.ptf.batch.error.PTFBatchBrinExtractionErrorHandler
```

### Return Codes

| Code | Meaning | Action |
|------|---------|--------|
| `0` | Success | Batch completed successfully |
| `1` | Non-critical error | Batch completed with warnings, system continues |
| `2` | Critical error | Batch failed, system stops |

### Gateway Timeout Handling

If the API returns a **504 Gateway Timeout**:

1. APIBatch waits 60 seconds
2. Polls the status endpoint with `JOB_ID` header
3. Checks job status every 5 minutes (300 seconds)
4. **Status values**:
   - `Active` = Keep polling
   - `Completed` = Return 0 (success)
   - `Failed` = Return 2 (error)

**Implementation**:

```java
private int handleGatewayIssue(int returnValue) {
  Thread.sleep(60000);  // Wait 1 minute
  
  while(status.equals("Active")) {
    Thread.sleep(300000);  // Wait 5 minutes
    status = getSitiApiBatchJobStatus(jobIdentifier);
  }
  
  return (status.equals("Completed")) ? 0 : 2;
}
```

---

## Dependencies & Technologies

### Build Tool
- **Maven** 3.6+: Build automation and dependency management

### Java Version
- **Java 8** (source and target: 1.8)

### Core Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| **spring-security-oauth2** | 2.0.12.RELEASE | OAuth2 authentication |
| **spring-web** | 4.3.12.RELEASE | HTTP client (RestTemplate) |
| **spring-context** | 4.3.12.RELEASE | Spring dependency injection |
| **spring-boot-starter-jdbc** | 2.3.0.RELEASE | JDBC support |
| **jackson-databind** | 2.10.4 | JSON serialization |
| **commons-lang3** | 3.0 | Utility classes (StringUtils) |
| **log4j-to-slf4j** | 2.17.1 | Logging bridge |

### Testing Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| **junit** | 4.13.2 | Unit testing framework |
| **mockito-core** | 2.21.0 | Mocking framework |
| **spring-test** | 5.1.2.RELEASE | Spring testing support |

### Code Coverage
- **JaCoCo** Maven plugin: Generates code coverage reports

### Project Dependencies (Internal)

| Module | Version | Purpose |
|--------|---------|---------|
| **siti_ptf_common** | 6.0.185-SNAPSHOT | Common utilities |
| **siti_ptf_domain** | 6.0.185-SNAPSHOT | Domain models |

---

## How to Run

### Prerequisites

1. **Java 8** installed
2. **Maven 3.6+** installed
3. **PTF Backend services** running (Configuration, Extraction, Position Integration APIs)
4. **SG-Connect OAuth2 server** accessible
5. Configuration files in place

### Step 1: Build the Module

```bash
cd siti_ptf_batch
mvn clean package
```

**Output**: 
- `target/siti_ptf_batch-6.0.185-SNAPSHOT.jar` - Executable JAR
- `target/siti_ptf_batch-6.0.185-SNAPSHOT.zip` - Assembly with scripts and configs

### Step 2: Extract Assembly (Optional)

```bash
unzip target/siti_ptf_batch-6.0.185-SNAPSHOT.zip
```

**Contents**:
```
siti_ptf_batch-6.0.185-SNAPSHOT/
├── bin/                    # All dependencies JAR files
├── config/                 # Configuration files
├── scripts/                # Shell scripts and SQL scripts
└── logs/                   # Log directory
```

### Step 3: Configure Properties

Edit `batch.properties` with your environment:

```properties
batch.ptfApiHost=your-server.com              # Your server
batch.clientId=your-client-id                  # Your OAuth2 client
batch.clientSecret=your-client-secret          # Your OAuth2 secret
batch.sgConnectAccessTokenURI=https://your-sg-connect-server/oauth2/access_token
```

### Step 4: Run a Batch

```bash
# Using JAR directly
java -jar target/siti_ptf_batch-6.0.185-SNAPSHOT.jar \
  src/main/resources/PTFBatchBrinExtraction.properties

# Or from the extracted assembly
cd siti_ptf_batch-6.0.185-SNAPSHOT
java -cp "bin/*:." com.socgen.siti.ptf.batch.client.APIBatch \
  config/PTFBatchBrinExtraction.properties
```

### Step 5: Check Exit Code

```bash
echo $?  # 0 = success, 1 = warning, 2 = error
```

### Example: Run JobRunner Batch

```bash
java -jar siti_ptf_batch.jar jobrunner.properties
```

### Example: Run with Additional Parameters

Some batches accept additional parameters:

```bash
java -jar siti_ptf_batch.jar \
  PTFBatchFluxValorisateur.properties \
  param1 \
  param2 \
  param3
```

---

## Testing

### Unit Tests Location

`src/test/java/com/socgen/siti/ptf/batch/client/`

### Test Classes

**APIBatchTest.java**:

```java
@RunWith(MockitoJUnitRunner.Silent.class)
public class APIBatchTest {
  @InjectMocks
  private APIBatch apiBatch;

  @Test
  public void testMain_FasBatch() {
    List<String> params = Arrays.asList(
      "src/main/resources/PTFBatchFluxValorisateur.properties", 
      "SGLB", "1", "1", "20210506"
    );
    assertNotNull(apiBatch.execute(params));
  }

  @Test
  public void testMain_JobRunner() {
    List<String> params = Arrays.asList(
      "src/main/resources/jobrunner.properties"
    );
    assertNotNull(apiBatch.execute(params));
  }
}
```

### Running Tests

```bash
# Run all tests
mvn test

# Run specific test
mvn test -Dtest=APIBatchTest

# Run with code coverage
mvn clean test jacoco:report
```

### Code Coverage Report

After running tests:

```bash
# Report location
target/site/jacoco/index.html
```

---

## Key Observations & Notes

### Important Points for New Developers

#### 1. **OAuth2 Token Caching**

The `SgConnectAccessTokenServiceImpl` caches tokens in a **static variable**:

```java
private static OAuth2AccessToken accessToken;
```

This means:
- Tokens are shared across all batch instances in the same JVM
- Token is regenerated only if expired or null
- If your token duration is 1 hour, the same token can be reused for multiple batches

⚠️ **Note**: System.out.println is used in token validation (see line 28 of SgConnectAccessTokenServiceImpl) - this violates the coding guidelines which prefer SLF4j logging.

#### 2. **Job Unique Identifier**

Each batch run generates a unique ID:

```java
jobIdentifier = 
  batchContext.getEntityId() +        // e.g., "6"
  batchContext.getUserId() +          // e.g., "ACCBATCH"
  remainingApiURI +                   // e.g., "/api/v1/extractions/brin/..."
  Math.abs(random.nextInt()) +        // Random number
  formatDateTime                      // Timestamp
```

This ID is:
- Used in status polling for long-running jobs
- Passed in the `JOB_ID` header when checking status
- Critical for identifying which batch you're checking the status of

#### 3. **Configuration File Hierarchy**

1. **First**: Global `batch.properties` loaded
2. **Second**: Batch-specific properties loaded (overrides global)
3. **Result**: Batch-specific settings override global settings

This allows customization per batch while sharing common settings.

#### 4. **Error Handler Instantiation**

Error handlers are instantiated **dynamically** by class name:

```java
String handlerClassName = properties.get("BATCH_ERROR_HANDLER");
this.batchHandler = (IBatchHandler) Class.forName(handlerClassName).newInstance();
```

**Implication**: 
- The error handler class must exist on the classpath
- Must have a default constructor (no-arg)
- If class not found, falls back to `DefaultBatchErrorHandler`

#### 5. **Port Resolution**

Different batches use different API ports:

```java
private String getPtfApiPort(String batchPropertyName) {
  switch(BatchPropertyNameEnum.retrieveBatchApiCategory(batchPropertyName)) {
    case CONFIGURATION:  return ptfConfigurationApiPort;     // 8087
    case EXTRACTION:     return ptfExtractionApiPort;         // 8088
    case POSITION_INTEGRATION: return ptfPositionIntegrationApiPort; // 8086
  }
}
```

This is determined by the batch type, not by properties - the mapping is in `BatchPropertyNameEnum`.

#### 6. **REST Template Configuration**

The code uses basic `RestTemplate` without custom configuration:

```java
RestTemplate restTemplate = new RestTemplate();
```

**Issues**:
- No connection timeout configured (can hang indefinitely)
- No socket timeout
- Limited error handling for network issues

**Better approach** (not implemented):

```java
SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
factory.setConnectTimeout(30000);  // 30 seconds
factory.setReadTimeout(30000);
RestTemplate restTemplate = new RestTemplate(factory);
```

#### 7. **Gateway Timeout Recovery**

The gateway timeout handling has a potential issue:

```java
Thread.sleep(60000);  // 1 minute
status = getSitiApiBatchJobStatus(jobIdentifier);
// Then polls every 5 minutes indefinitely
```

**Concern**: No maximum retry count or timeout for overall polling. Could run forever if API status endpoint fails.

#### 8. **Error Handler Counter**

Some error handlers (like `PTFBatchBrinExtractionErrorHandler`) maintain a **static counter**:

```java
private static int mCounter = 0;
```

**Issue**: In multi-threaded environments or if same handler is reused, this could cause race conditions.

#### 9. **Status Response Parsing**

The status polling uses string manipulation to extract status:

```java
String compare = status.substring(status.indexOf(":\"")+2, status.indexOf("\"}"));
```

**Issue**: 
- Fragile - depends on exact JSON format
- No JSON parsing (could use Jackson's ObjectMapper)
- If format changes, parsing will fail

#### 10. **Logging**

Uses **SLF4j** with **Log4j** backing:

```properties
# log4j.properties
log4j.rootLogger=INFO, consoleAppender
log4j.appender.consoleAppender=org.apache.log4j.ConsoleAppender
```

**Note**: The code has one violation - uses `System.out.println` in SgConnectAccessTokenServiceImpl (line 28).

#### 11. **Missing Implementation**

PTFFlowIntegration.properties file is incomplete:

```properties
# BATCH_ERROR_HANDLER line is missing!
BATCH_TYPE = SIMPLE_CALL
BATCH_ID = 1
API_SCOPE = api.ptf-position-integration.v1
REMAINING_API_URI = /api/v1/jobrunner/process-flow-integration
# ERROR HANDLER NOT DEFINED
```

When this batch runs, it will fall back to `DefaultBatchErrorHandler`.

#### 12. **SQL Scripts for Maintenance**

The `src/main/resources/sqlScripts/` directory contains SQL scripts for:

- Archival operations
- View materialization
- Data cleanup
- VM (Materialized View) refreshes
- Job control

These are not executed by Java code but likely run separately for maintenance.

---

### Potential Issues & Improvements

| Issue | Severity | Recommendation |
|-------|----------|-----------------|
| String parsing of JSON status | Medium | Use Jackson ObjectMapper for JSON parsing |
| No connection timeout | Medium | Configure RestTemplate with timeouts |
| Static token variable | Low | Consider thread-safe token refresh mechanism |
| Static counter in handlers | Low | Use instance variables instead |
| System.out.println usage | Low | Replace with SLF4j logger |
| No retry limit for polling | High | Add max retry count to gateway timeout recovery |
| Incomplete PTFFlowIntegration.properties | Low | Add BATCH_ERROR_HANDLER configuration |
| No input validation | Medium | Validate configuration values before use |

---

### Architecture Alignment with Clean Architecture

This module follows the **Clean Architecture** principles:

| Layer | What's Here | Purpose |
|-------|-----------|---------|
| **Infra (Framework)** | `APIBatch`, `JobRunnerClient`, `PtfFlowIntegrationClient` | Entry points & REST API communication |
| **Infra (Controllers/Adapters)** | Error handlers | Adapt errors for external systems |
| **Domain** | `BatchContext` | Value objects holding batch data |
| **Service** | Not present | Batch module is mostly infra layer |

**Dependency Flow** (correct direction):

```
Infra Layer (This Module)
    ↓
Service Layer (siti_ptf_common, siti_ptf_domain)
    ↓
Domain Layer
```

This is correct - batch module depends on common and domain modules, not the other way around.

---

### File Size Reference

| File | Lines | Purpose |
|------|-------|---------|
| `APIBatch.java` | 464 | Main batch executor |
| `JobRunnerClient.java` | 306 | JobRunner-specific batch |
| `PtfFlowIntegrationClient.java` | 273 | Flow integration batch |
| `BatchContext.java` | 93 | Context holder |
| `SgConnectAccessTokenServiceImpl.java` | 76 | OAuth2 token service |
| `BatchPropertyNameEnum.java` | 78 | Configuration mapping |
| `IBatchHandler.java` | 84 | Error handler interface |
| `DefaultBatchErrorHandler.java` | 105 | Default error handler |

---

## Summary

The **siti_ptf_batch** module is a sophisticated batch processing framework that:

1. **Loads configuration** from property files
2. **Authenticates** with OAuth2 (SG-Connect)
3. **Calls PTF microservices** via REST APIs
4. **Handles errors** with pluggable strategies
5. **Supports multiple batch types** (data extraction, position updates, etc.)
6. **Manages long-running operations** with status polling
7. **Returns proper exit codes** for external schedulers

It serves as the bridge between external job schedulers and the PTF backend microservices, providing a robust, configurable batch execution framework that handles authentication, error management, and asynchronous operation tracking.


