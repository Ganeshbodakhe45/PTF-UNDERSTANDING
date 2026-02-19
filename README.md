# SITI PTF Complete Knowledge Transfer Guide

**Single Comprehensive Document for Complete Understanding**  

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture Overview](#architecture-overview)
3. [System Architecture Diagram](#system-architecture-diagram)
4. [Module Structure](#module-structure)
5. [Key Classes & Components](#key-classes--components)
6. [REST API Reference](#rest-api-reference)
7. [Database Overview](#database-overview)
8. [Configuration & Environment](#configuration--environment)
9. [Development Guidelines](#development-guidelines)
10. [Common Tasks & How-Tos](#common-tasks--how-tos)
11. [Troubleshooting & FAQs](#troubleshooting--faqs)
12. [4-Week Onboarding Schedule](#4-week-onboarding-schedule)

---

## Project Overview

### What is SITI PTF?

**SITI PTF** (Modernization of Portfolio System) is a backend microservices system that manages financial portfolios, securities positions, and trading operations for Société Générale.

### Purpose

- Manage portfolio positions and securities
- Process mass loading of position data
- Generate portfolio reports and inventories
- Support trading and risk management
- Provide REST APIs for portfolio queries
- Handle batch processing for bulk operations

### Technology Stack

| Component | Technology |
|-----------|-----------|
| Language | Java 8+ |
| Framework | Spring Boot |
| Build Tool | Maven 3.6.0+ |
| Database | Oracle 12.2+ |
| ORM | Spring Data JPA (Hibernate) |
| Mapping | MapStruct |
| Security | Spring Security + OAuth2/OIDC |
| Logging | SLF4J + Logback |
| Testing | JUnit 5 + Mockito + AssertJ |
| API Docs | OpenAPI 3.0 + Swagger |
| DB Versioning | Liquibase |

---

## Architecture Overview

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT APPLICATIONS                       │
│                  (Web, Mobile, Excel, Reports)              │
└───────────────────────┬────────────────────────────────────┘
                        │ HTTPS/REST
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   ┌─────────────┐  ┌──────────────┐  ┌──────────────┐
   │  OAuth2     │  │ Search &     │  │  Position    │
   │  Proxy API  │  │  Consult API │  │  Massloading │
   │ (Port 8081) │  │ (Port 8080)  │  │     API      │
   └──────┬──────┘  └──────┬───────┘  └──────┬───────┘
          │                │                  │
    ┌─────┴────────────────┴──────────────────┴─────────┐
    │                                                    │
    │          ┌─────────────────────────────────┐      │
    │          │    Spring Boot Microservices   │      │
    │          │                                 │      │
    │  ┌───────┼─────────────────────────────────┼────┐ │
    │  │       └─────────────────────────────────┘    │ │
    │  │                                               │ │
    │  │  ┌──────────────────────────────────────┐   │ │
    │  │  │      Service Layer (Business Logic) │   │ │
    │  │  │  - SearchService                    │   │ │
    │  │  │  - PositionMassloadingService       │   │ │
    │  │  │  - PortfolioGroupService            │   │ │
    │  │  └──────────────────────────────────────┘   │ │
    │  │                                               │ │
    │  │  ┌──────────────────────────────────────┐   │ │
    │  │  │   Domain Layer (Populators, DTOs)   │   │ │
    │  │  │  - SecurityPopulator                │   │ │
    │  │  │  - PartyPopulator                   │   │ │
    │  │  │  - Position DTOs                    │   │ │
    │  │  └──────────────────────────────────────┘   │ │
    │  │                                               │ │
    │  │  ┌──────────────────────────────────────┐   │ │
    │  │  │    Infra Layer (Data Access, WS)    │   │ │
    │  │  │  - JPA Repositories                 │   │ │
    │  │  │  - DAO Implementations              │   │ │
    │  │  │  - REST Clients                     │   │ │
    │  │  └──────────────────────────────────────┘   │ │
    │  └──────────────────────────────────────────────┘ │
    └─────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │   Oracle     │  │  SG-Connect  │  │  Message     │
   │  Database    │  │  OAuth2      │  │  Broker      │
   │              │  │  Server      │  │  (Kafka/RMQ) │
   └──────────────┘  └──────────────┘  └──────────────┘
```

### Clean Architecture Principles

This project follows **Clean Architecture** with clear separation:

- **Domain Layer** (`siti_ptf_domain`): Pure business logic, no framework knowledge
- **Service Layer**: Orchestration and use cases
- **Infrastructure Layer**: Framework code, database, external services
- **API/Presentation Layer**: REST controllers and DTOs

**Dependency Rule**: Code can only depend on layers below it:
```
Domain → Service → Infra → API (no upward dependencies)
```

---

## System Architecture Diagram

### Module Dependency Graph

```
┌─────────────────┐
│  siti_ptf_      │
│  common         │ (Shared: Constants, Enums, Context)
└────────┬────────┘
         │
┌────────▼────────────────────────────────────────────┐
│          Domain Layer Modules                       │
│  ┌──────────────┐  ┌─────────────────────────┐    │
│  │ siti_ptf_    │  │ siti_ptf_report_domain  │    │
│  │ domain       │  │                         │    │
│  │              │  └─────────────────────────┘    │
│  │ - Populators │                                 │
│  │ - DTOs       │                                 │
│  │ - Validators │                                 │
│  └──────────────┘                                 │
└────────┬─────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────┐
│          Mapping Layer                              │
│  ┌──────────────┐  ┌──────────────────────────┐   │
│  │ siti_ptf_    │  │ siti_ptf_report         │   │
│  │ mapper       │  │ (Jasper, PDF generation)│   │
│  └──────────────┘  └──────────────────────────┘   │
└────────┬─────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────┐
│          Integration Layer                          │
│  ┌──────────────────┐  ┌────────────────────────┐  │
│  │ siti_ptf_ws_     │  │ siti_ptf_ws_route     │  │
│  │ client           │  │ siti_ptf_route_api    │  │
│  └──────────────────┘  └────────────────────────┘  │
└────────┬─────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────┐
│      API & Application Layer (10+ Services)        │
│  - ptf-proxy-api-service (8081)                    │
│  - ptf-search-consult (8080)                       │
│  - ptf-position-massloading                        │
│  - ptf-portfolio-group                             │
│  - ptf-message-consumer                            │
│  - ptf-configuration                               │
│  - ptf-extraction                                  │
│  - ptf-integration-report (8082)                   │
│  - safe-url-service                                │
│  - siti_ptf_batch                                  │
└────────┬─────────────────────────────────────────┘
         │
┌────────▼─────────────────────────────────────────────┐
│  External Systems Integration                        │
│  - Oracle Database                                   │
│  - SG-Connect (OAuth2)                               │
│  - Message Broker (Kafka/RabbitMQ)                   │
│  - External Web Services (BDR, SIR, SIA)             │
│  - Azure Data Lake Storage                           │
└────────────────────────────────────────────────────┘
```

---

## Module Structure

### Core Modules (14 Total)

| Module | Purpose | Type | Responsibilities |
|--------|---------|------|------------------|
| **siti_ptf_common** | Shared utilities | Library | Constants, enums, context, utilities |
| **siti_ptf_domain** | Domain objects | Library | Populators, DTOs, business logic |
| **siti_ptf_mapper** | DTO mapping | Library | MapStruct mappers, DTO definitions |
| **siti_ptf_report_domain** | Report domain | Library | Report data structures |
| **siti_ptf_report** | Report generation | Library | Jasper reports, PDF generation |
| **siti_ptf_ws_client** | External clients | Library | SOAP/REST client stubs |
| **siti_ptf_ws_route** | Message routing | Library | Message routing config |
| **siti_ptf_route_api** | API routing | Library | API routing config |
| **ptf-proxy-api-service** | Token proxy | Service (8081) | OAuth2 token validation |
| **ptf-search-consult** | Portfolio search | Service (8080) | Position search & reporting |
| **ptf-position-massloading** | Bulk loading | Service/Batch | Position loading & validation |
| **ptf-portfolio-group** | Pool management | Service | Portfolio grouping |
| **ptf-message-consumer** | Message listener | Consumer | Message processing |
| **siti_ptf_batch** | Batch runner | Batch | Batch job execution |

### Dependency Order

```
siti_ptf_common (base)
    ↓
siti_ptf_domain + siti_ptf_report_domain
    ↓
siti_ptf_mapper + siti_ptf_report
    ↓
siti_ptf_ws_client + siti_ptf_ws_route + siti_ptf_route_api
    ↓
Service Modules (APIs & Batch)
```

---

## Key Classes & Components

### 1. Request Context Management

```java
// Represents the current user/client context
ApplicationUserContext {
  - userId: String
  - userScopes: Set<String>
  - userPermissions: Set<String>
  - accountingDate: LocalDate
}

// Represents the request context (user + application data)
ApplicationRequestContext extends ApplicationUserContext {
  - batchInputDTO: BatchInputDTO
  - executionContext: ExecutionContext
}

// Factory to create contexts
DefaultRequestContextFactory.createContext(...)
```

### 2. Populator Framework

Populators enrich and validate domain objects. They follow a chain pattern:

```java
ITimeValuedPopulator interface {
  void populate(Object sourceObject, Object targetObject);
}

// Examples:
- SecurityPopulator: enriches security objects
- SecurityCodificationPopulator: adds security codes
- PartyPopulator: enriches party/entity information
- RealEstatePopulator: enriches real estate positions
- OTCPositionPopulator: enriches OTC derivatives

// Usage Pattern:
SecurityPopulator populator = new SecurityPopulator(...);
populator.populate(sourceDTO, targetDomain);
// Now targetDomain is enriched with additional data
```

### 3. Service Layer

Convention: Services have **no interface**, only implementation.

```java
@Service
public class SearchService {
  @Autowired
  private IPositionDAO positionDAO;
  
  @Autowired
  private PositionMapper mapper;

  public List<PositionDTO> findByPortfolio(final String portfolioCode) {
    final List<Position> positions = positionDAO.findByPortfolioCode(portfolioCode);
    return mapper.toDTOList(positions);
  }
}
```

### 4. REST Controllers

Controllers must:
- Use `@PreAuthorize` for security
- Document with OpenAPI annotations
- Return `ResponseEntity<>` for flexibility
- Use SLF4J logger (never System.out)
- Use meaningful error responses

```java
@RestController
@RequestMapping("/api/v1/ptf/searchAndConsultResult")
@PreAuthorize("hasAuthority('SCOPE_api.ptf-search-consult.v1')")
public class SearchAndConsultController {
  
  private static final Logger logger = LoggerFactory.getLogger(SearchAndConsultController.class);
  
  @PostMapping("/positions")
  @Operation(
    summary = "Search positions",
    security = @SecurityRequirement(name = "sgconnect")
  )
  public ResponseEntity<InventorySearchResultDTO> searchPositions(
    @RequestBody final InventorySearchCriteriaDTO criteria) {
    
    logger.debug("Searching positions with criteria: {}", criteria);
    // Validate input
    // Call service
    // Return response
    return ResponseEntity.ok(result);
  }
}
```

### 5. Exception Handling

```
PtfException (base)
  ├── PtfSQLException (database errors)
  ├── PtfInteropCriticalApplicationException (critical failures)
  ├── PtfInteropWarningApplicationException (recoverable)
  └── Custom exceptions
```

---

## REST API Reference

### Overview

All APIs are secured with **OAuth2**, use **REST** principles, and return **JSON**.

### Service 1: PTF Search & Consult API

**Base URL**: `http://localhost:8080/api/v1/ptf/searchAndConsultResult`

| Endpoint | Method | Purpose | Input | Output |
|----------|--------|---------|-------|--------|
| `/positions` | POST | Search positions by criteria | InventorySearchCriteriaDTO | InventorySearchResultDTO |
| `/consolidatedInventory` | POST | Get consolidated inventory | Criteria | PtfConsolidatedInventoryResponse |
| `/unitaryInventory` | POST | Get unitary inventory | Criteria | PtfUnitaryInventoryResponse |
| `/evolution` | POST | Get portfolio evolution | Criteria | EvolutionPortfolioDTO[] |
| `/export` | POST | Export to Excel/PDF | Criteria | Binary File |
| `/certifiedInventory` | POST | Get certified inventory | Criteria | CertifiedInventoryDTO |

#### Example: Search Positions

**Request**:
```bash
curl -X POST http://localhost:8080/api/v1/ptf/searchAndConsultResult/positions \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "portfolioCode": "PTF001",
    "accountId": "ACC123",
    "pageNumber": 1,
    "pageSize": 50
  }'
```

**Response** (200 OK):
```json
{
  "totalCount": 250,
  "pageNumber": 1,
  "pageSize": 50,
  "positions": [
    {
      "positionId": "POS001",
      "portfolioCode": "PTF001",
      "accountId": "ACC123",
      "instrumentCode": "ISIN_123",
      "instrumentLabel": "Government Bond 2030",
      "quantity": 1000,
      "currencyCode": "EUR",
      "marketValue": 102500,
      "costValue": 100000,
      "pnl": 2500
    }
  ]
}
```

### Service 2: PTF Position Mass-Loading API

**Base URL**: `http://localhost:8080/api/v1/ptf/positionMassloading`

| Endpoint | Method | Purpose | Input |
|----------|--------|---------|-------|
| `/positions/load` | POST | Load positions from Excel file | MultipartFile |
| `/validate` | POST | Validate positions | PositionValidationDTO[] |
| `/job/{jobId}/status` | GET | Check job status | Path param |
| `/job/{jobId}/result` | GET | Get job result | Path param |
| `/job/{jobId}/cancel` | POST | Cancel job | Path param |
| `/job/{jobId}/errorReport.xlsx` | GET | Download errors | Path param |

#### Example: Load Positions

**Request**:
```bash
curl -X POST http://localhost:8080/api/v1/ptf/positionMassloading/positions/load \
  -H "Authorization: Bearer TOKEN" \
  -F "file=@positions.xlsx" \
  -F "portfolioCode=PTF001" \
  -F "accountId=ACC123"
```

**Response** (202 Accepted):
```json
{
  "jobId": "JOB_20260219_001",
  "status": "IN_PROGRESS",
  "message": "Position loading started",
  "checkStatusUrl": "/api/v1/ptf/positionMassloading/job/JOB_20260219_001/status"
}
```

### Service 3: PTF Portfolio Group API

**Base URL**: `http://localhost:8080/api/v1/ptf/portfolioPool`

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/pools` | GET | List all pools |
| `/pool/{id}` | GET | Get pool details |
| `/pool` | POST | Create pool |
| `/pool/{id}` | PUT | Update pool |
| `/pool/{id}` | DELETE | Delete pool |

### Service 4: PTF Proxy API Service

**Base URL**: `http://localhost:8081/api/v1/ptf/ptfProxyService`

| Endpoint | Method | Purpose | Input |
|----------|--------|---------|-------|
| `/token` | POST | Get OAuth2 access token | Grant type, scopes |
| `/validate` | POST | Validate JWT token | Token |

---

## Database Overview

### Core Tables

```sql
-- Portfolio related
PTF_PORTFOLIO (code, label, manager_id, ...)
PTF_ACCOUNT (id, portfolio_id, account_type, ...)
PTF_PORTFOLIO_POOL (pool_code, pool_name, pool_type, ...)

-- Positions
PTF_POSITION (id, portfolio_id, account_id, instrument_id, quantity, ...)
PTF_SECURITY_POSITION (extends PTF_POSITION)
PTF_CASH_POSITION (extends PTF_POSITION)
PTF_REAL_ESTATE_POSITION (extends PTF_POSITION)

-- Instruments
PTF_INSTRUMENT (id, isin, code, label, family, ...)
PTF_SECURITY (security_id, market_code, sector, rating, ...)
PTF_OTC_DERIVATIVE (otc_id, derivative_type, underlying_id, ...)
PTF_LISTED_DERIVATIVE (listed_deriv_id, derivative_type, ...)

-- Party/Entity
PTF_PARTY (id, code, label, type, ...)

-- Batch Processing
PTF_BATCH_JOB (job_id, job_name, status, start_time, end_time, ...)
PTF_BATCH_ERROR (error_id, job_id, error_line_number, error_code, ...)
```

### Data Model - Portfolio Hierarchy

```
PTF_PORTFOLIO (Portfolio Master)
├─ Code: PTF001
├─ Label: Growth Fund
├─ Manager: Asset Mgmt Team
│
├── PTF_ACCOUNT (Safe-Keeping/Cash Account)
│   ├─ Code: ACC001
│   ├─ Type: SAFE_KEEPING
│   │
│   └── PTF_POSITION (Holding/Security)
│       ├─ ISIN: XXXXXX
│       ├─ Quantity: 1000
│       ├─ Market Value: 102,500
│       └─ As Of Date: 2026-02-19
│           │
│           └── PTF_INSTRUMENT (Bond, Stock, etc.)
│               ├─ ISIN: XXXXXX
│               ├─ Label: Gov Bond
│               └─ Family: BOND
│
└── PTF_PORTFOLIO_POOL (Portfolio Grouping)
    ├─ Pool: Global Equity Pool
    ├─ Type: EXPLICIT
    └─ Members: [PTF001, PTF002, PTF003]
```

### Key Queries

**Get All Positions for Portfolio on Date**:
```sql
SELECT p.*, i.INSTRUMENT_LABEL, a.ACCOUNT_CODE
FROM PTF_POSITION p
JOIN PTF_INSTRUMENT i ON p.INSTRUMENT_ID = i.INSTRUMENT_ID
JOIN PTF_ACCOUNT a ON p.ACCOUNT_ID = a.ACCOUNT_ID
JOIN PTF_PORTFOLIO pf ON p.PORTFOLIO_ID = pf.PORTFOLIO_ID
WHERE pf.PORTFOLIO_CODE = 'PTF001'
  AND p.AS_OF_DATE = TRUNC(SYSDATE)
ORDER BY a.ACCOUNT_CODE, i.INSTRUMENT_LABEL;
```

**Get Consolidated View by Family and Account**:
```sql
SELECT 
  a.ACCOUNT_CODE,
  pfa.INSTRUMENT_FAMILY,
  COUNT(*) as POSITION_COUNT,
  SUM(p.QUANTITY) as TOTAL_QUANTITY,
  SUM(p.MARKET_VALUE) as TOTAL_MARKET_VALUE
FROM PTF_POSITION p
JOIN PTF_POSITION_BY_FAMILY_ACCOUNT pfa ON p.POSITION_ID = pfa.POSITION_ID
JOIN PTF_ACCOUNT a ON p.ACCOUNT_ID = a.ACCOUNT_ID
WHERE p.PORTFOLIO_ID = :portfolioId
  AND p.AS_OF_DATE = :asOfDate
GROUP BY a.ACCOUNT_CODE, pfa.INSTRUMENT_FAMILY
ORDER BY a.ACCOUNT_CODE, pfa.INSTRUMENT_FAMILY;
```

---

## Configuration & Environment

### Application Properties

**Location**: `ptf-search-consult/src/main/resources/application.properties`

```properties
# Spring Boot
spring.application.name=ptf-search-consult
spring.profiles.active=dev

# Server
server.port=8080
server.servlet.context-path=/

# Database
spring.datasource.url=${ptf.ds.url}
spring.datasource.username=${ptf.ds.user}
spring.datasource.password=${ptf.ds.password}
spring.datasource.driver-class-name=oracle.jdbc.driver.OracleDriver

# JPA/Hibernate
spring.jpa.database-platform=org.hibernate.dialect.Oracle12cDialect
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false

# Logging
logging.level.root=INFO
logging.level.com.socgen.siti.ptf=DEBUG
logging.file.name=logs/ptf-search-consult.log
```

### Environment-Specific Configuration

**File**: `config/env_dev.properties`

```properties
# Database
ptf.ds.url=jdbc:oracle:thin:@//dbbdevdb0123.fr.world.socgen:1522/PTFZD01
ptf.ds.user=PTF$OWNER
ptf.ds.password=<ENCRYPTED>

# OAuth2/SG-Connect
sgconnect.root.uri=https://sgconnect-hom.fr.world.socgen/sgconnect/oauth2
sgconnect.accesstoken.uri=https://sgconnect-hom.fr.world.socgen/sgconnect/oauth2/access_token
ptf.client.id=205a38bf-42c0-4a80-954e-2aca41370225
ptf.client.secret=<ENCRYPTED>

# Server Ports
siti.ptf.api.safeservice.port=8080
siti.ptf.api.integrationreport.port=8082

# External Services
ws.bdr.host.socgen=ngpuatweb01.dns43.socgen
ws.bdr.port.socgen=7010
ws.sir.host.socgen=sisuatweb01.dns43.socgen
ws.sir.port.socgen=7010
```

### Secrets Management

**Never commit secrets!** Use environment variables:

```bash
# Linux/Mac
export PTF_DS_PASSWORD="actual_password"
export PTF_CLIENT_SECRET="actual_secret"

# Windows
set PTF_DS_PASSWORD=actual_password
set PTF_CLIENT_SECRET=actual_secret

# In properties
ptf.ds.password=${env.PTF_DS_PASSWORD}
ptf.client.secret=${env.PTF_CLIENT_SECRET}
```

---

## Development Guidelines

### Code Formatting Standards

**Required by Eclipse Code Formatter**:

- Use **2 spaces** for indentation (not tabs)
- Opening braces on **new line** for classes/methods
- Spaces after commas, before braces
- Line length limit: **140 characters**
- One blank line between methods

**Example - Good**:
```java
public class PortfolioService
{
  private final PortfolioRepository repository;

  public Optional<Portfolio> findById(final Long id)
  {
    return repository.findById(id);
  }
}
```

**Example - Bad**:
```java
public class PortfolioService {
    public Optional<Portfolio> findById(final Long id) {
        return repository.findById(id);
    }
}
```

### Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Classes | PascalCase | `PositionDTO`, `SearchService` |
| Methods/Variables | camelCase | `findByCode`, `portfolioId` |
| Constants | UPPER_SNAKE_CASE | `MAX_BATCH_SIZE` |
| DTOs | Suffix with `Dto` | `PositionDto` |
| Folders | kebab-case | `ptf-search-consult` |

### Imports

- **No star imports** (`import java.util.*`)
- Order:
  1. Static imports (other)
  2. `java.*`
  3. `javax.*`
  4. `jakarta.*`
  5. `org.*`
  6. `com.*`
  7. Other imports

### Method Requirements

Every method must have:
- `final` parameters
- Meaningful variable names (no abbreviations)
- Early returns for validation
- Proper exception handling
- Logging for important operations

**Good Example**:
```java
public Optional<Portfolio> findPortfolioByCode(final String portfolioCode)
{
  if (Objects.isNull(portfolioCode)) {
    logger.warn("Portfolio code is null");
    return Optional.empty();
  }

  final Optional<Portfolio> portfolio = repository.findByCode(portfolioCode);
  if (portfolio.isPresent()) {
    logger.debug("Found portfolio: {}", portfolioCode);
  }
  return portfolio;
}
```

### Testing Requirements

Every new code must have tests. Use **JUnit 5** + **Mockito** + **AssertJ**:

```java
@RunWith(MockitoJUnitRunner.class)
class PortfolioServiceTest
{
  @InjectMocks
  private PortfolioService service;

  @Mock
  private PortfolioRepository repository;

  @Test
  void find_by_code_should_return_portfolio_when_exists()
  {
    // Arrange
    final Portfolio portfolio = new Portfolio("PTF001", "Test");
    when(repository.findByCode("PTF001")).thenReturn(Optional.of(portfolio));

    // Act
    final Optional<Portfolio> result = service.findByCode("PTF001");

    // Assert
    assertThat(result).isPresent().contains(portfolio);
  }
}
```

### Logging Standards

Use **SLF4J** (never `System.out`):

```java
private static final Logger logger = LoggerFactory.getLogger(MyClass.class);

logger.debug("Debug message: {}", value);
logger.info("Important info: {}", value);
logger.warn("Warning: {}", value);
logger.error("Error occurred", exception);
```

### Transaction Management

- Use `@Transactional` **only on DAOs**, not services
- Use `readOnly = true` for queries
- Keep transactions **small and focused**

```java
// In DAO/Repository
@Repository
public class PositionDAOImpl implements IPositionDAO
{
  @Transactional(readOnly = true)
  public Optional<Position> findById(final Long id)
  {
    return repository.findById(id);
  }

  @Transactional
  public Position save(final Position position)
  {
    return repository.save(position);
  }
}

// In Service - NO @Transactional
@Service
public class PortfolioService
{
  @Autowired
  private IPositionDAO positionDAO;

  public Portfolio createPortfolio(final CreatePortfolioRequest request)
  {
    final Position position = positionDAO.save(...);
    return portfolio;
  }
}
```

---

## Common Tasks & How-Tos

### Task 1: Add a New REST Endpoint

**Step 1**: Create DTO
```java
@Data
public class MyRequestDto
{
  private String portfolioCode;
  private LocalDate date;
}

@Data
public class MyResponseDto
{
  private String result;
  private List<String> warnings;
}
```

**Step 2**: Create service method
```java
@Service
public class MyBusinessService
{
  public MyResponseDto processRequest(final MyRequestDto request)
  {
    if (Objects.isNull(request.getPortfolioCode())) {
      throw new IllegalArgumentException("Portfolio code required");
    }

    final MyResponseDto response = new MyResponseDto();
    // ... business logic
    return response;
  }
}
```

**Step 3**: Add controller endpoint
```java
@RestController
@RequestMapping(value = "/api/v1/ptf/my", produces = APPLICATION_JSON_VALUE)
@PreAuthorize("hasAuthority('SCOPE_api.ptf-my.v1')")
public class MyController
{
  @Autowired
  private MyBusinessService service;

  @PostMapping("/process")
  @Operation(
    summary = "Process my request",
    security = @SecurityRequirement(name = "sgconnect")
  )
  public ResponseEntity<MyResponseDto> process(
    @RequestBody final MyRequestDto request)
  {
    final MyResponseDto response = service.processRequest(request);
    return ResponseEntity.ok(response);
  }
}
```

### Task 2: Add a Database Query

**Step 1**: Create DAO interface
```java
public interface IPositionDAO
{
  List<Position> findByPortfolioCode(String portfolioCode);
}
```

**Step 2**: Implement in JPA
```java
@Repository
public class PositionDAOImpl implements IPositionDAO
{
  @Autowired
  private PositionRepository repository;

  @Transactional(readOnly = true)
  public List<Position> findByPortfolioCode(final String portfolioCode)
  {
    return repository.findByPortfolioCodeOrderByCode(portfolioCode);
  }
}

@Repository
public interface PositionRepository extends JpaRepository<Position, Long>
{
  List<Position> findByPortfolioCodeOrderByCode(String portfolioCode);
}
```

### Task 3: Write Unit Tests

```java
@SpringBootTest
@AutoConfigureMockMvc
class MyControllerTest
{
  @Autowired
  private MockMvc mockMvc;

  @MockBean
  private MyBusinessService service;

  @Test
  @WithSgConnectClient(scopes = {"api.ptf-my.v1"})
  void process_should_return_success() throws Exception
  {
    final MyResponseDto response = new MyResponseDto("OK", List.of());
    when(service.processRequest(any())).thenReturn(response);

    mockMvc.perform(post("/api/v1/ptf/my/process")
      .contentType(APPLICATION_JSON)
      .content(asJson(new MyRequestDto("PTF001", LocalDate.now()))))
      .andExpect(status().isOk());
  }
}
```

### Task 4: Add Database Column

**Step 1**: Create Liquibase migration
```yaml
# 05_add_column_status_to_position.yaml
databaseChangeLog:
  - changeSet:
      id: 5
      author: team
      changes:
        - addColumn:
            tableName: PTF_POSITION
            columns:
              - column:
                  name: POSITION_STATUS
                  type: VARCHAR2(20)
                  defaultValue: ACTIVE
                  constraints:
                    nullable: false
```

**Step 2**: Update JPA Entity
```java
@Entity
@Table(name = "PTF_POSITION")
public class Position
{
  // ... existing fields ...

  @Column(name = "POSITION_STATUS", nullable = false, length = 20)
  private String status;
}
```

**Step 3**: Apply migration
```bash
mvn liquibase:update
```

---

## Troubleshooting & FAQs

### FAQ 1: How do I run the project locally?

**Steps**:
1. Clone repository: `git clone <url>`
2. Install: Java 8+, Maven 3.6.0+
3. Build: `mvn clean install`
4. Run: `mvn spring-boot:run -pl ptf-search-consult`
5. Verify: `curl http://localhost:8080/actuator/health`

### FAQ 2: How do I debug position import failure?

**Steps**:
1. Get job ID from response
2. Check status: `GET /job/{jobId}/status`
3. Get errors: `GET /job/{jobId}/result`
4. Query database:
   ```sql
   SELECT * FROM PTF_BATCH_ERROR 
   WHERE JOB_ID = 'JOB_20260219_001'
   ORDER BY ERROR_LINE_NUMBER;
   ```

**Common Errors**:
- `INVALID_SECURITY` - ISIN not found
- `INVALID_QUANTITY` - Non-numeric or negative
- `INVALID_ACCOUNT` - Account doesn't exist
- `DUPLICATE_POSITION` - Position already exists

### FAQ 3: How do I handle 403 Forbidden?

**Error**: Access Denied

**Solution**:
1. Check token has correct scope: `SCOPE_api.ptf-search-consult.v1`
2. Verify endpoint `@PreAuthorize` annotation
3. Get new token with correct scopes
4. Check token expiration

### FAQ 4: How do I test API endpoints?

**Using cURL**:
```bash
TOKEN=$(curl -s -X POST http://localhost:8081/api/v1/ptf/ptfProxyService/token \
  -H "Content-Type: application/json" \
  -d '{"grantType":"client_credentials"}' | jq -r '.accessToken')

curl -X POST http://localhost:8080/api/v1/ptf/searchAndConsultResult/positions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"portfolioCode":"PTF001","pageSize":10}'
```

**Using Integration Tests**:
```java
@SpringBootTest
@AutoConfigureMockMvc
class SearchControllerIT
{
  @Autowired
  private MockMvc mockMvc;

  @Test
  @WithSgConnectClient(scopes = {"api.ptf-search-consult.v1"})
  void should_search_positions() throws Exception
  {
    mockMvc.perform(post("/api/v1/ptf/searchAndConsultResult/positions")
      .contentType(APPLICATION_JSON)
      .content("{\"portfolioCode\":\"PTF001\"}"))
      .andExpect(status().isOk());
  }
}
```

### FAQ 5: Database connection fails - what to check?

**Checklist**:
1. Connection string: `ptf.ds.url` in properties
2. Credentials: `ptf.ds.user` and `ptf.ds.password`
3. Test: `sqlplus -l user/password@dbhost:port/schema`
4. Firewall: `ping` the database host
5. Oracle driver version matches

### FAQ 6: How do I write better unit tests?

**Best Practices**:
- Test method name: `{methodName}_should_{expectedResult}_when_{condition}`
- Arrange-Act-Assert pattern
- Mock dependencies with Mockito
- Use AssertJ for fluent assertions
- Test success AND failure cases
- Keep tests focused and isolated

### FAQ 7: Transaction rolled back - why?

**Check**:
1. Exception in logs
2. `@Transactional` only on DAO, not service
3. Constraint violations (foreign key, unique)
4. Deadlocks: `SELECT * FROM v$lock;`
5. Transaction isolation level

### FAQ 8: How do I add logging?

```java
// Good
private static final Logger logger = LoggerFactory.getLogger(MyClass.class);
logger.debug("Debug: {}", value);
logger.info("Info: {}", value);
logger.warn("Warning: {}", value);
logger.error("Error", exception);

// Bad
System.out.println("Debug"); // Never use!
```

### FAQ 9: Performance is slow - what to check?

1. Enable SQL logging: `spring.jpa.show-sql=true`
2. Look for N+1 queries
3. Add pagination (don't fetch all)
4. Use proper indices
5. Check execution plan: `EXPLAIN PLAN FOR SELECT ...`

### FAQ 10: Project won't compile - what to do?

**Checklist**:
1. Run: `mvn clean install`
2. Check imports (no star imports!)
3. Check for syntax errors
4. Did you update parent module? Rebuild children
5. Missing dependency? Add to pom.xml
6. Wrong Java version? Set `maven.compiler.source=1.8`

---

## 4-Week Onboarding Schedule

### Week 1: Setup & Understanding

**Day 1-2**:
- [ ] Read this complete guide
- [ ] Clone repository
- [ ] Install Java 8+, Maven, IDE
- [ ] Setup code formatter (eclipse-code-style.xml)

**Day 3-5**:
- [ ] Build project: `mvn clean install`
- [ ] Run tests: `mvn test`
- [ ] Start service: `mvn spring-boot:run -pl ptf-search-consult`
- [ ] Test API: `curl http://localhost:8080/actuator/health`
- [ ] Read architecture section (this document)

**Goal**: Understand overall architecture

### Week 2: Deep Dive & Code Exploration

**Day 1-3**:
- [ ] Read Module Structure section
- [ ] Trace 1 complete API flow
- [ ] Review 5 existing classes
- [ ] Understand populator pattern
- [ ] Understand service layer pattern

**Day 4-5**:
- [ ] Write 2 unit tests
- [ ] Fix 1 small bug (typo, log message)
- [ ] Run full test suite
- [ ] Check code coverage: `mvn jacoco:report`

**Goal**: Understand code patterns and structure

### Week 3: Hands-On Development

**Day 1-2**:
- [ ] Add one small feature (validation rule, log statement)
- [ ] Add database query
- [ ] Write integration test
- [ ] Create git branch

**Day 3-5**:
- [ ] Make pull request
- [ ] Get code review feedback
- [ ] Fix review comments
- [ ] Merge to main

**Goal**: Make first meaningful contribution

### Week 4: Productivity & Integration

**Day 1-5**:
- [ ] Develop real feature end-to-end
- [ ] Understand batch jobs
- [ ] Debug complex issues
- [ ] Help other developers
- [ ] Continuous learning

**Goal**: Become productive team member

---

## Quick Reference Commands

```bash
# Build & Test
mvn clean install                     # Full build
mvn test                              # Run tests
mvn test -Dtest=ClassName            # Specific test
mvn clean test jacoco:report          # With coverage
mvn sonar:sonar                       # SonarQube analysis

# Run Services
mvn spring-boot:run -pl ptf-search-consult
mvn -f ptf-position-massloading/pom.xml spring-boot:run

# Database
mvn liquibase:update                  # Apply migrations
mvn liquibase:status                  # Check status
mvn liquibase:rollback -Dliquibase.rollbackCount=1

# Code Quality
mvn checkstyle:check                  # Check style
mvn formatter:format                  # Format code

# Dependencies
mvn dependency:tree                   # Show tree
mvn dependency:resolve-plugins        # Resolve plugins
```

---

## Key Principles to Remember

### ✅ Do This

- Keep it simple (KISS)
- Write tests first (TDD)
- Code for readability
- Use descriptive names
- Keep functions small
- Handle errors properly
- Document complex logic
- Ask questions early
- Separation of Concerns
- Layered Architecture
- Testability
- Clean Code

### ❌ Don't Do This

- Database queries in controllers
- Business logic in REST layer
- Skip null checks
- Commit secrets
- Make huge methods
- Use star imports
- Ignore SonarQube warnings
- Mix concerns (domain + infra)
- Use System.out.println
- Premature optimization

---

## Success Criteria

Your onboarding is successful when you can:

- ✅ Explain the 3-layer architecture
- ✅ Identify which module to modify for a feature
- ✅ Write unit tests with proper mocking
- ✅ Add a new REST endpoint
- ✅ Write database queries using repositories
- ✅ Debug a failing test
- ✅ Understand error messages and stack traces
- ✅ Follow code formatting standards
- ✅ Create a feature branch and pull request
- ✅ Contribute meaningfully to the codebase

**Timeline**: 2-4 weeks for junior developers

---

## Getting Help

### Before Asking
1. Search in this document (Ctrl+F)
2. Check FAQ section
3. Look at actual code examples
4. Review similar implementations
5. Try to solve yourself first

### Getting Help
- Architecture questions → Senior developer or architect
- Code questions → Code review team
- Database questions → Database admin
- Build issues → Maven expert
- Deployment questions → DevOps team

### Expected Response Times
- Code review: Same day
- Critical issues: Within 1 hour
- Architecture questions: Within 2 hours
- General questions: Within 24 hours

---

## Final Notes

### Welcome to the Team! 🚀

This guide contains everything a new developer needs to know about SITI PTF.

**Remember**:
- Take your time to understand
- Ask questions early
- Write tests for everything
- Follow the patterns
- Help others learn
- Keep improving

### Next Steps

1. Read this document completely (2-3 hours)
2. Clone the repository
3. Run it locally
4. Explore the code
5. Write your first test
6. Make your first contribution

### Contact & Support

- **Questions?** Ask your mentor/senior developer
- **Stuck?** Check the FAQ section of this document
- **Found issue?** Report it or fix it
- **Suggestions?** Share with the team

---

**Version**: 1.0 (Complete)  
**Date**: February 19, 2026  
**Status**: ✅ Ready for Use  

**Welcome aboard! Good luck with your SITI PTF journey!** 🎊


