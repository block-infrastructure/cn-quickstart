# Canton Network Application - Project Generation Template

This document is a comprehensive reference for an AI agent to understand how a Canton Network (CN) application is structured, so it can generate a brand new project. It separates **infrastructure you consume** from **application code you write**, covering local development, Docker orchestration, and DevNet deployment.

---

## Table of Contents

1. [Three-Layer Architecture](#1-three-layer-architecture)
2. [What You Write vs What You Consume](#2-what-you-write-vs-what-you-consume)
3. [Project Directory Structure](#3-project-directory-structure)
4. [Layer 1: Canton Infrastructure (Consumed)](#4-layer-1-canton-infrastructure-consumed)
5. [Layer 2: Splice / Canton Network Services (Consumed)](#5-layer-2-splice--canton-network-services-consumed)
6. [Layer 3: Your Application (Written)](#6-layer-3-your-application-written)
7. [Daml Smart Contracts](#7-daml-smart-contracts)
8. [Spring Boot Backend](#8-spring-boot-backend)
9. [React Frontend](#9-react-frontend)
10. [OpenAPI Specification (Shared Contract)](#10-openapi-specification-shared-contract)
11. [Build System](#11-build-system)
12. [Docker Compose Orchestration](#12-docker-compose-orchestration)
13. [Authentication Architecture](#13-authentication-architecture)
14. [Onboarding and Bootstrap Sequence](#14-onboarding-and-bootstrap-sequence)
15. [PQS (Projected Query Service) Integration](#15-pqs-projected-query-service-integration)
16. [Token Standard Integration](#16-token-standard-integration)
17. [DevNet Deployment](#17-devnet-deployment)
18. [Environment Configuration](#18-environment-configuration)
19. [Port Conventions](#19-port-conventions)
20. [Reusable Code Patterns](#20-reusable-code-patterns)
21. [Generating a New Project Checklist](#21-generating-a-new-project-checklist)

---

## 1. Three-Layer Architecture

Every Canton Network application has three distinct layers. Understanding which layer each component belongs to is essential.

```
+------------------------------------------------------------------+
|  LAYER 3: YOUR APPLICATION (you write this)                      |
|  - React Frontend (served by nginx)                              |
|  - Spring Boot Backend (Java 21)                                 |
|  - Daml Smart Contracts (.daml files)                            |
|  - OpenAPI Specification (shared between frontend & backend)     |
|  - Onboarding Scripts (register tenants, create users)           |
+------------------------------------------------------------------+
        |                       |                       |
        | HTTP/REST             | gRPC (Ledger API)     | HTTP (Token APIs)
        v                       v                       v
+------------------------------------------------------------------+
|  LAYER 2: SPLICE (Canton Network Services) - consumed            |
|  - Validator Apps (one per participant, manages wallets)          |
|  - Scan App (token registry, network metadata)                   |
|  - SV App (super validator governance)                           |
|  - Wallet Web UIs                                                |
+------------------------------------------------------------------+
        |                       |
        | connects to           | manages
        v                       v
+------------------------------------------------------------------+
|  LAYER 1: CANTON (Distributed Ledger Infrastructure) - consumed  |
|  - Participant Nodes (Ledger API, Admin API)                     |
|  - Sequencer (transaction ordering)                              |
|  - Mediator (transaction validation)                             |
|  - PostgreSQL (storage for all components)                       |
+------------------------------------------------------------------+
```

**Key insight**: Canton and Splice are infrastructure you consume via APIs. Your application interacts with them through:
- **Ledger API** (gRPC) on Canton participants - for submitting commands (writes)
- **PQS** (SQL/JDBC) - for reading contract state (reads)
- **Validator Admin API** (HTTP) on Splice validators - for party/wallet management
- **Token Registry API** (HTTP) on Splice scan app - for token operations

---

## 2. What You Write vs What You Consume

### You Write (Application Layer)

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Daml contracts | Daml 3.x | Business logic as smart contracts |
| Backend service | Spring Boot (Java 21) | REST API, ledger interaction, PQS queries |
| Frontend | React + TypeScript + Vite | User interface |
| OpenAPI spec | YAML | Shared API contract (generates code for both sides) |
| Nginx config | nginx.conf templates | Reverse proxy + static file serving |
| Onboarding scripts | Shell scripts | Bootstrap tenants, register users |
| Docker Compose (app) | YAML | Your services + references to infra modules |

### You Consume (Infrastructure)

| Component | Image Source | Purpose |
|-----------|-------------|---------|
| Canton | `ghcr.io/.../canton:{version}` | Participant nodes, sequencer, mediator |
| Splice | `ghcr.io/.../splice-app:{version}` | Validator apps, wallet, token standard |
| PQS (Scribe) | `europe-docker.pkg.dev/.../scribe:{version}` | Streams ledger state to PostgreSQL |
| PostgreSQL | `postgres:14` | Shared database for all components |
| Keycloak | `quay.io/keycloak/keycloak:{version}` | OAuth2/OIDC identity provider (optional) |
| Observability | Grafana/Prometheus/Loki/Tempo | Monitoring (optional) |
| Wallet/ANS Web UIs | Pre-built Splice containers | Wallet and naming service UIs |

---

## 3. Project Directory Structure

A new project should follow this structure. Items marked `[INFRA]` are consumed as-is; items marked `[APP]` are what you write.

```
my-canton-app/
├── .env                                    # [APP] Base env vars (versions, ports)
├── .env.local                              # [APP] Generated by setup (party hint, auth mode)
├── compose.yaml                            # [APP] Root Docker Compose (your services)
├── Makefile                                # [APP] Build orchestration
├── build.gradle.kts                        # [APP] Root Gradle config
│
├── daml/                                   # [APP] Smart contracts
│   ├── build.gradle.kts                    # Daml build + Java codegen
│   ├── my-contracts/
│   │   ├── daml.yaml                       # SDK version, package name, data-dependencies
│   │   └── daml/MyApp/
│   │       ├── MyTemplate.daml             # Your contract templates
│   │       └── Util.daml                   # Helpers
│   ├── my-contracts-tests/                 # Daml Script tests
│   │   ├── daml.yaml
│   │   └── daml/MyApp/Tests.daml
│   └── dars/                               # Splice token standard DARs (data-dependencies)
│       ├── splice-api-token-metadata-v1-1.0.0.dar
│       ├── splice-api-token-holding-v1-1.0.0.dar
│       ├── splice-api-token-allocation-v1-1.0.0.dar
│       └── splice-api-token-allocation-request-v1-1.0.0.dar
│
├── backend/                                # [APP] Spring Boot backend
│   ├── build.gradle.kts                    # Plugins: spring-boot, openapi-generator, protobuf
│   └── src/main/java/com/myorg/myapp/
│       ├── App.java                        # @SpringBootApplication entry point
│       ├── config/
│       │   ├── LedgerConfig.java           # Ledger host/port/appId config
│       │   ├── PostgresConfig.java         # PQS datasource config
│       │   └── SecurityConfig.java         # Common security beans
│       ├── ledger/
│       │   ├── LedgerApi.java              # gRPC client for Canton Ledger API [REUSABLE]
│       │   └── TokenStandardProxy.java     # HTTP client for Splice token APIs [REUSABLE]
│       ├── pqs/
│       │   └── Pqs.java                    # JDBC client for PQS queries [REUSABLE]
│       ├── repository/
│       │   ├── DamlRepository.java         # Domain-specific PQS queries [APP-SPECIFIC]
│       │   └── TenantPropertiesRepository.java  # Multi-tenant registry [REUSABLE]
│       ├── security/
│       │   ├── Auth.java                   # Enum: OAUTH2 | SHARED_SECRET [REUSABLE]
│       │   ├── AuthUtils.java              # Auth helper (asAdminParty, asAuthenticatedParty) [REUSABLE]
│       │   ├── AuthenticatedPartyProvider.java  # Interface [REUSABLE]
│       │   ├── AuthenticatedUserProvider.java   # Interface [REUSABLE]
│       │   ├── TokenProvider.java          # Interface [REUSABLE]
│       │   ├── sharedsecret/
│       │   │   └── SharedSecretConfig.java # Spring Security for dev mode [REUSABLE]
│       │   └── oauth2/
│       │       ├── OAuth2Config.java       # Spring Security for production [REUSABLE]
│       │       └── OAuth2AuthenticationSuccessHandler.java
│       ├── service/
│       │   ├── MyTemplateApiImpl.java      # Implements generated controller interface [APP-SPECIFIC]
│       │   ├── AdminApiImpl.java           # Tenant registration [REUSABLE]
│       │   ├── UserApiImpl.java            # Authenticated user info [REUSABLE]
│       │   └── FeatureFlagsApiImpl.java    # Auth mode flags [REUSABLE]
│       └── utility/
│           └── TracingUtils.java           # OpenTelemetry helpers
│
├── frontend/                               # [APP] React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── src/
│       ├── App.tsx                         # Router + Context providers
│       ├── api.ts                          # OpenAPI client (from shared spec)
│       ├── types.ts                        # Additional TypeScript types
│       ├── views/
│       │   ├── LoginView.tsx               # Auth mode-aware login [REUSABLE]
│       │   ├── HomeView.tsx                # Protected landing page
│       │   └── MyFeatureView.tsx           # Domain-specific views [APP-SPECIFIC]
│       ├── stores/
│       │   ├── userStore.tsx               # Auth state management [REUSABLE]
│       │   ├── toastStore.tsx              # Notifications [REUSABLE]
│       │   └── myFeatureStore.tsx          # Domain state [APP-SPECIFIC]
│       ├── components/
│       │   ├── Header.tsx                  # Nav bar with user info + wallet link
│       │   ├── Modal.tsx                   # Generic modal wrapper
│       │   └── ToastNotification.tsx       # Toast UI
│       └── utils/
│           ├── commandId.ts               # Unique command ID generation
│           ├── format.ts                   # Date/amount formatting
│           └── error.ts                    # Error handling HOC
│
├── common/                                 # [APP] Shared between frontend & backend
│   └── openapi.yaml                        # OpenAPI 3.0 specification
│
├── config/                                 # [APP] Runtime configuration
│   └── nginx/
│       ├── frontend.conf                   # Nginx virtual host template
│       └── common-backend-proxy-settings.conf
│
├── docker/                                 # Docker setup
│   ├── modules/                            # [INFRA] Consumed infrastructure modules
│   │   ├── localnet/                       # Full local Canton + Splice stack
│   │   │   ├── compose.yaml               # Canton, Splice, postgres, nginx services
│   │   │   ├── compose.env                # Image versions, profiles
│   │   │   ├── conf/
│   │   │   │   ├── canton/                # Participant config (ports, DB, auth)
│   │   │   │   │   ├── app.conf           # Shared participant template
│   │   │   │   │   ├── app-provider/      # Per-participant overrides
│   │   │   │   │   ├── app-user/
│   │   │   │   │   └── sv/
│   │   │   │   └── splice/                # Validator config (party hints, wallet users)
│   │   │   └── env/                       # Environment files (ports, DB creds)
│   │   │
│   │   ├── devnet/                         # DevNet deployment (external validator)
│   │   │   ├── compose.yaml               # Keycloak, PQS, backend, nginx stubs
│   │   │   ├── compose.env
│   │   │   ├── start.sh                   # DevNet startup script
│   │   │   ├── stop.sh
│   │   │   ├── env/                       # DevNet-specific env vars
│   │   │   ├── onboarding/               # DevNet bootstrap scripts
│   │   │   └── conf/                     # Keycloak realm config
│   │   │
│   │   ├── splice-onboarding/             # [INFRA] Bootstrap container
│   │   │   ├── compose.yaml
│   │   │   └── docker/
│   │   │       ├── entrypoint.sh          # Container lifecycle manager
│   │   │       ├── utils.sh               # Canton API utilities
│   │   │       ├── app-provider.sh        # Discover provider party
│   │   │       └── app-user.sh            # Discover user party
│   │   │
│   │   ├── pqs/                           # [INFRA] PQS/Scribe containers
│   │   │   ├── compose.yaml
│   │   │   └── env/
│   │   │
│   │   ├── keycloak/                      # [INFRA] OAuth2 identity provider
│   │   │   ├── compose.yaml
│   │   │   └── conf/                     # Realm definitions
│   │   │
│   │   └── observability/                 # [INFRA] Monitoring stack (optional)
│   │       ├── compose.yaml
│   │       └── conf/                     # Grafana dashboards, Prometheus config
│   │
│   ├── backend-service/                   # [APP] Backend Docker config
│   │   ├── start.sh                       # Container entrypoint
│   │   ├── env/
│   │   │   └── app.env                    # Backend env vars
│   │   └── onboarding/
│   │       ├── onboarding.sh              # Create Canton user for backend
│   │       └── env/
│   │           ├── shared-secret.env
│   │           └── oauth2.env
│   │
│   └── register-app-user-tenant/          # [APP] Tenant registration scripts
│       ├── shared-secret.sh
│       ├── oauth2.sh
│       └── env/
│           ├── shared-secret.env
│           └── oauth2.env
│
└── integration-test/                      # [APP] E2E tests (Playwright)
    ├── package.json
    └── tests/
```

---

## 4. Layer 1: Canton Infrastructure (Consumed)

Canton is the distributed ledger runtime. You do not write Canton code. You consume it as Docker images.

### What Canton Provides

| Component | Purpose | API |
|-----------|---------|-----|
| Participant Node | Hosts your parties, processes transactions | Ledger API (gRPC, ports `X901`) |
| Admin API | Node administration | HTTP (ports `X902`) |
| JSON API | REST wrapper for Ledger API | HTTP (ports `X975`) |
| Sequencer | Orders transactions across participants | Internal Canton protocol |
| Mediator | Validates multi-party transactions | Internal Canton protocol |

### Participant Configuration Pattern

Each participant is defined in `.conf` files with a port prefix:

| Participant | Prefix | Ledger API | Admin API | JSON API |
|------------|--------|-----------|-----------|----------|
| app-user | `2` | 2901 | 2902 | 2975 |
| app-provider | `3` | 3901 | 3902 | 3975 |
| sv | `4` | 4901 | 4902 | 4975 |

Canton config uses a shared template:
```hocon
canton.participants.app-provider = ${_participant} {
  storage.config.properties.databaseName = participant-app-provider
  ledger-api.port = 3901
  admin-api.port = 3902
  http-ledger-api.port = 3975
}
```

### Party IDs

Party IDs have two parts: `<party-hint>::<namespace>`

```
app_provider_quickstart-mycompany-1::1220abcdef1234567890...
\_________________________________/   \________________________/
  You provide this                     Participant generates this
  (human-readable hint)               (crypto key fingerprint)
```

Your application does not create parties. The Splice validator creates them during startup. Your onboarding scripts discover them by querying the participant's user API.

### Canton Users (Not Human Users)

Canton users are access control records on a participant node:
- Created via `POST /v2/users`
- Have `primaryParty` (the party they represent)
- Have rights: `CanActAs`, `CanReadAs`, `ParticipantAdmin`
- Multiple Canton users can act as the same party
- NOT the same as human login identities

Your backend authenticates to the Ledger API as a Canton user, then submits commands on behalf of the party.

### Resource Requirements (LocalNet)

```yaml
canton:     4GB RAM (JVM: 512m-2560m)
postgres:   2GB RAM
```

---

## 5. Layer 2: Splice / Canton Network Services (Consumed)

Splice provides Canton Network-specific services on top of Canton.

### What Splice Provides

| Component | Purpose | API |
|-----------|---------|-----|
| Validator App | Manages party, wallet, token operations for a participant | Validator Admin API (ports `X903`) |
| Scan App | Token registry, network metadata | Token Registry API (port 5012) |
| SV App | Super validator governance | SV Admin API (port 5014) |
| Wallet Web UI | Token management UI for end users | HTTP (served via nginx) |

### How Splice Connects to Canton

```
Splice Validator ──connects to──> Canton Participant
       │
       ├── Allocates parties (POST /v2/parties)
       ├── Creates Canton users (POST /v2/users)
       ├── Manages wallet holding contracts
       └── Handles token standard operations
```

### Token Standard Contracts

Canton doesn't store "balances". Tokens are Daml contracts:

| Contract Type | Purpose |
|--------------|---------|
| HoldingV1 | Tokens owned by a party |
| AllocationV1 | Committed payment (tokens locked for transfer) |
| AllocationRequestV1 | Request for a party to approve a payment |
| MetadataV1 | Registry metadata about the token instrument |

### The AllocationRequest Interface Pattern

This is the key integration point between your app and the token standard. When your contract implements `AllocationRequest`, the user's wallet automatically detects it as a pending payment:

```
Your App Contract (implements AllocationRequest)
        │
        ├── Wallet sees it as pending payment request
        ├── User approves in wallet → creates Allocation contract
        └── Your app exercises Allocation_ExecuteTransfer → tokens move
```

### Resource Requirements (LocalNet)

```yaml
splice:         3GB RAM (JVM: 512m-2560m)
wallet-web-ui:  256MB RAM each
```

---

## 6. Layer 3: Your Application (Written)

This is what you actually build. Everything else is infrastructure.

### Application Services in Docker Compose

Your root `compose.yaml` defines three services:

1. **backend-service** - Your Spring Boot application
   - Image: `eclipse-temurin:21-jdk`
   - Mounts: compiled backend JAR, onboarding volume, OTEL agent
   - Depends on: PQS (started), splice-onboarding (healthy)

2. **register-app-user-tenant** - Init container for tenant registration
   - Extends: splice-onboarding container
   - Runs once, then exits
   - Calls backend API to register the app-user tenant

3. **nginx** - Frontend + reverse proxy
   - Mounts: compiled React app from `frontend/dist/`
   - Proxies `/api/*` to backend service
   - Depends on: backend-service (started)

### Read vs Write Pattern

This is the fundamental data flow pattern:

| Operation | Data Source | Protocol | Why |
|-----------|-----------|----------|-----|
| Read contracts | PQS (PostgreSQL) | SQL over JDBC | Fast, supports JOINs, filtered by party |
| Write/exercise | Canton Participant | gRPC (Ledger API) | Authoritative writes with Daml validation |
| Token operations | Splice Scan App | HTTP REST | External service for allocation metadata |

```
Frontend ──HTTP/REST──> Backend ──gRPC──> Canton Participant (writes)
                           │
                           └──JDBC/SQL──> PQS PostgreSQL (reads)
                           │
                           └──HTTP──> Splice Token Registry (token ops)
```

---

## 7. Daml Smart Contracts

### Configuration (daml.yaml)

```yaml
sdk-version: 3.4.10
name: my-app-contracts
source: daml
version: 0.0.1
dependencies:
  - daml-prim
  - daml-stdlib
data-dependencies:
  - ../dars/splice-api-token-metadata-v1-1.0.0.dar
  - ../dars/splice-api-token-holding-v1-1.0.0.dar
  - ../dars/splice-api-token-allocation-v1-1.0.0.dar
  - ../dars/splice-api-token-allocation-request-v1-1.0.0.dar
build-options:
  - --ghc-option=-Wunused-binds
  - --ghc-option=-Wunused-matches
```

The `data-dependencies` on Splice token standard DARs are required for any app that integrates with token payments.

### Contract Design Conventions

- Template names: `UpperCamelCase` (e.g., `License`, `AppInstall`)
- Choice names: `TemplateName_ChoiceName` (e.g., `License_Renew`, `AppInstall_Cancel`)
- Fields: `lowerCamelCase`
- Use `signatory` for parties that must authorize contract creation
- Use `observer` for parties that can see but not modify
- Use `ensure` for contract invariants

### Common Patterns from the Example App

**Request/Accept Pattern** (two-step authorization):
```daml
-- Step 1: One party creates a request
template MyRequest
  with
    requester : Party
    approver : Party
    details : Text
  where
    signatory requester
    observer approver
    choice MyRequest_Accept : ContractId MyContract
      with meta : Metadata
      controller approver
      do create MyContract with ..

-- Step 2: Other party accepts, creating the actual contract
template MyContract
  with
    requester : Party
    approver : Party
  where
    signatory requester, approver
```

**Fee Payment via AllocationRequest Interface**:
```daml
template MyPaymentRequest
  with
    payer : Party
    receiver : Party
    amount : Decimal
    instrumentId : InstrumentId
  where
    signatory payer, receiver

    -- Implement AllocationRequest so wallets detect this as a payment request
    interface instance AllocationRequest for MyPaymentRequest where
      view = AllocationRequestView with
        receiver = receiver
        settlement = SettlementInfo with
          executor = receiver
          -- ... settlement details
        transferLegs = [TransferLeg with
          sender = payer
          receiver = receiver
          amount = amount
          instrumentId = instrumentId]
```

**Nonconsuming Choices** (contract stays active):
```daml
nonconsuming choice MyTemplate_Query : SomeResult
  with actor : Party
  controller actor
  do -- read-only or creates child contracts
```

### Daml Testing

Tests use Daml Script in a separate package:
```yaml
# daml/my-contracts-tests/daml.yaml
sdk-version: 3.4.10
name: my-contracts-tests
version: 0.0.1
dependencies:
  - daml-prim
  - daml-stdlib
  - daml-script
data-dependencies:
  - ../../daml/my-contracts/.daml/dist/my-app-contracts-0.0.1.dar
```

---

## 8. Spring Boot Backend

### Entry Point

```java
@SpringBootApplication(exclude = {SecurityAutoConfiguration.class})
@ConfigurationPropertiesScan("com.myorg.myapp")
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

Excludes default Spring Security auto-configuration because custom auth configs (shared-secret or OAuth2) are provided via Spring profiles.

### LedgerApi.java (Reusable)

The core gRPC client for Canton Ledger API:

```java
@Component
public class LedgerApi {
    // Manages gRPC connection to Canton participant
    // Injects Bearer token via interceptor
    // Uses Transcode for DTO <-> Protobuf conversion

    // Create a new contract
    public <T extends Template> CompletableFuture<Void> create(T entity, String commandId)

    // Exercise a choice and get the typed result
    public <T extends Template, Result, C extends Choice<T, Result>>
    CompletableFuture<Result> exerciseAndGetResult(
        ContractId<T> contractId, C choice, String commandId,
        List<DisclosedContract> disclosedContracts)

    // Submit arbitrary commands
    public CompletableFuture<SubmitResponse> submitCommands(
        List<Command> cmds, String commandId,
        List<DisclosedContract> disclosedContracts)
}
```

Key features:
- Token-based authentication via `TokenProvider` interface
- Automatic tracing via `@WithSpan` (OpenTelemetry)
- Converts `ListenableFuture` (gRPC) to `CompletableFuture` (Java async)
- Supports disclosed contracts for cross-party references

### Pqs.java (Reusable)

The JDBC client for PQS (read-side queries):

```java
@Component
public class Pqs {
    private final JdbcTemplate jdbcTemplate;

    // Get all active contracts of a template type
    public <T extends Template> CompletableFuture<List<Contract<T>>> active(Class<T> clazz)

    // Get active contracts with WHERE clause
    public <T extends Template> CompletableFuture<List<Contract<T>>> activeWhere(
        Class<T> clazz, String whereClause, Object... params)

    // Get single contract by ID
    public <T extends Template> CompletableFuture<Optional<Contract<T>>> contractByContractId(
        Class<T> clazz, Object... params)

    // Custom SQL query
    public CompletableFuture<Void> query(String sql, RowCallbackHandler callback, Object... params)
}
```

Uses PQS SQL functions:
- `active('PackageName:ModuleName:TemplateName')` - returns all active contracts
- Contract payloads stored as JSON in PostgreSQL
- Supports JOINs across template types via JSON operators (`->`, `->>`)

### TokenStandardProxy.java (Reusable)

HTTP client for Splice token registry:

```java
@Component
public class TokenStandardProxy {
    // Get the registry admin party ID (needed for InstrumentId)
    public CompletableFuture<String> getRegistryAdminId()

    // Get transfer context for completing a token allocation
    public CompletableFuture<Optional<ChoiceContext>> getAllocationTransferContext(String allocationId)
}
```

Uses OpenAPI-generated Java clients for the token metadata and allocation APIs.

### Service Implementation Pattern

Services implement OpenAPI-generated controller interfaces:

```java
@Controller
@RequestMapping("${openapi.asset.base-path:}")
public class MyTemplateApiImpl implements MyTemplateApi {

    private final AuthUtils auth;
    private final LedgerApi ledgerApi;
    private final Pqs pqs;

    @Override
    public CompletableFuture<ResponseEntity<List<MyTemplateDto>>> listMyTemplates() {
        return auth.asAuthenticatedParty(party ->
            pqs.activeWhere(MyTemplate.class, "payload->>'owner' = ?", party)
                .thenApply(contracts -> contracts.stream()
                    .map(this::toDto)
                    .toList())
                .thenApply(ResponseEntity::ok)
        );
    }

    @Override
    public CompletableFuture<ResponseEntity<MyResultDto>> exerciseMyChoice(
            String contractId, String commandId, MyChoiceRequest request) {
        return auth.asAuthenticatedParty(party ->
            ledgerApi.exerciseAndGetResult(
                new ContractId<>(contractId),
                new MyTemplate_MyChoice(request.getParam()),
                commandId,
                List.of()
            ).thenApply(result -> ResponseEntity.status(201).body(toDto(result)))
        );
    }
}
```

### DamlRepository.java (Application-Specific)

Domain-specific queries that JOIN multiple contract types:

```java
@Repository
public class DamlRepository {
    private final Pqs pqs;

    public CompletableFuture<List<MyAggregateView>> findMyAggregates(String party) {
        String sql = """
            SELECT parent.contract_id AS parent_cid,
                   parent.payload AS parent_payload,
                   child.contract_id AS child_cid,
                   child.payload AS child_payload
            FROM active(?) parent
            LEFT JOIN active(?) child ON
                parent.payload->>'id' = child.payload->>'parentId'
            WHERE parent.payload->>'owner' = ?
            """;
        // Execute and aggregate results
    }
}
```

### Application Configuration (application.yml)

```yaml
server:
  port: ${BACKEND_PORT}
  error:
    include-message: always

ledger:
  host: ${LEDGER_HOST:localhost}
  port: ${LEDGER_PORT:6865}
  application-id: ${AUTH_APP_PROVIDER_BACKEND_USER_ID:AppId}
  registry-base-uri: ${REGISTRY_BASE_URI}

application:
  tenants:
    AppProvider:
      tenantId: AppProvider
      partyId: ${APP_PROVIDER_PARTY}
      internal: true   # internal = admin role

spring:
  datasource:
    url: jdbc:postgresql://${POSTGRES_HOST}:${POSTGRES_PORT:5432}/${POSTGRES_DATABASE}
    username: ${POSTGRES_USERNAME}
    password: ${POSTGRES_PASSWORD}
```

---

## 9. React Frontend

### API Client Setup

Uses `openapi-client-axios` to auto-generate typed API calls from the shared OpenAPI spec:

```typescript
// api.ts
import OpenAPIClientAxios from 'openapi-client-axios';
import openApi from '../../common/openapi.yaml';

const api: OpenAPIClientAxios = new OpenAPIClientAxios({
    definition: openApi as any,
    withServer: { url: '/api' },
});

api.init();
export default api;
```

### React Context Store Pattern

State management uses React Context with `useCallback` hooks:

```typescript
// stores/myFeatureStore.tsx
interface MyFeatureContextType {
    items: MyItem[];
    loading: boolean;
    fetchItems: () => Promise<void>;
    createItem: (request: CreateItemRequest) => Promise<void>;
}

export const MyFeatureProvider: React.FC<PropsWithChildren> = ({ children }) => {
    const [items, setItems] = useState<MyItem[]>([]);
    const { displayError, displaySuccess } = useToastStore();

    const fetchItems = useCallback(async () => {
        const client = await api.getClient();
        const response = await client.listMyItems();
        setItems(response.data);
    }, []);

    const createItem = useCallback(async (request: CreateItemRequest) => {
        const client = await api.getClient();
        const commandId = generateCommandId();
        await client.createMyItem(null, commandId, request);
        displaySuccess('Item created');
        await fetchItems();
    }, [fetchItems, displaySuccess]);

    return (
        <MyFeatureContext.Provider value={{ items, loading, fetchItems, createItem }}>
            {children}
        </MyFeatureContext.Provider>
    );
};
```

### Routing and Auth-Aware Views

```typescript
// App.tsx
<BrowserRouter>
    <UserProvider>
        <ToastProvider>
            <MyFeatureProvider>
                <Routes>
                    <Route path="/login" element={<LoginView />} />
                    <Route path="/" element={<HomeView />} />
                    <Route path="/my-feature" element={<MyFeatureView />} />
                </Routes>
            </MyFeatureProvider>
        </ToastProvider>
    </UserProvider>
</BrowserRouter>
```

### Login View (Auth Mode Aware)

The login view checks `/api/feature-flags` to determine auth mode:
- **shared-secret**: Renders a username form (no password)
- **oauth2**: Fetches `/api/login-links` and renders OAuth2 login buttons

### Utility Functions

```typescript
// commandId.ts - unique IDs for ledger commands
export function generateCommandId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}

// error.ts - higher-order error handler
export const withErrorHandling = (operationName: string) =>
    (fn: Function) => async (...args) => {
        try { return await fn(...args); }
        catch (error) { toast.displayError(`Error ${operationName}: ...`); throw error; }
    };
```

---

## 10. OpenAPI Specification (Shared Contract)

The OpenAPI spec at `common/openapi.yaml` is the single source of truth for the API contract. It generates:
- **Backend**: Spring controller interfaces (via `openapi-generator` Gradle plugin)
- **Frontend**: TypeScript types (via `openapi-client-axios` code generation)

### Standard Endpoint Patterns

```yaml
# Action endpoints use colon-suffix convention
/my-templates/{contractId}:my-action:
  post:
    operationId: exerciseMyAction
    parameters:
      - name: contractId
        in: path
      - name: commandId
        in: header
        description: Idempotent command identifier
    requestBody:
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/MyActionRequest'
    responses:
      '200':
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/MyActionResult'
```

### Standard Endpoint Groups

Every Canton app should define at minimum:

| Tag | Endpoints | Purpose |
|-----|-----------|---------|
| Feature Flags | `GET /feature-flags` | Auth mode, feature toggles |
| Auth | `GET /user`, `POST /login`, `POST /logout` | Authentication |
| Admin | `POST /admin/tenant-registrations` | Multi-tenant management |
| Domain-Specific | `GET/POST /my-templates/...` | Your business logic |

---

## 11. Build System

### Gradle Multi-Module Structure

```
Root build.gradle.kts
├── daml/ (sub-project)
│   └── build.gradle.kts
│       ├── Task: compileDaml (daml damlc build)
│       ├── Task: codeGen (Transcode Java bindings from DAR)
│       └── Task: testDaml (daml damlc test)
│
└── backend/ (sub-project)
    └── build.gradle.kts
        ├── Plugin: spring-boot
        ├── Plugin: openapi-generator (3 specs: app API + token metadata + allocation)
        ├── Plugin: protobuf (gRPC Ledger API stubs)
        ├── Task: openApiGenerate → Spring controller interfaces
        ├── Task: openApiGenerateMetadata → Token metadata client
        ├── Task: openApiGenerateAllocation → Allocation API client
        ├── Task: compileJava (depends on all codegen tasks)
        └── Task: downloadOtelAgent (OpenTelemetry Java agent)
```

### Code Generation Pipeline

```
1. Daml source  ──damlc build──>  DAR file
2. DAR file     ──Transcode──>    Java contract classes
3. OpenAPI spec ──generator──>    Spring interfaces + TS types
4. Proto files  ──protoc──>       gRPC Java stubs
```

### Makefile Targets

```makefile
# Setup
make setup              # Interactive first-time configuration
make install-daml-sdk   # Install Daml SDK

# Build
make build              # Full build (frontend + backend + daml + docker images)
make build-frontend     # npm install && npm run build
make build-backend      # ./gradlew :backend:build
make build-daml         # ./gradlew :daml:build distTar
make build-docker-images # docker compose build

# Run
make start              # Start full stack
make stop               # Stop full stack
make restart            # stop + start
make start-vite-dev     # Hot-reload frontend dev server

# Test
make test               # All tests
make test-daml          # Daml Script tests
make integration-test   # Playwright E2E tests

# Utility
make status             # Docker service status
make logs / make tail   # View logs
make canton-console     # Interactive Daml console
make shell              # Daml shell

# Cleanup
make clean              # Gradle clean
make clean-docker       # Docker down + remove volumes
make clean-all          # Everything
```

---

## 12. Docker Compose Orchestration

### Module Composition Strategy

The Makefile assembles Docker Compose from modular files:

```makefile
DOCKER_COMPOSE_FILES := -f compose.yaml                            # Your app services
DOCKER_COMPOSE_FILES += -f ${LOCALNET_DIR}/compose.yaml            # Canton + Splice
DOCKER_COMPOSE_FILES += -f ${MODULES_DIR}/splice-onboarding/compose.yaml  # Bootstrap
DOCKER_COMPOSE_FILES += -f ${MODULES_DIR}/pqs/compose.yaml         # PQS streaming
# Conditional:
DOCKER_COMPOSE_FILES += -f ${MODULES_DIR}/keycloak/compose.yaml    # if AUTH_MODE=oauth2
DOCKER_COMPOSE_FILES += -f ${MODULES_DIR}/observability/compose.yaml # if OBSERVABILITY_ENABLED
```

### Service Dependency Chain

```
postgres
    └── canton (depends_on: postgres healthy)
            └── splice (depends_on: canton healthy)
                    └── splice-onboarding (depends_on: splice healthy)
                            ├── pqs-app-provider (depends_on: splice-onboarding)
                            └── backend-service (depends_on: pqs + splice-onboarding)
                                    ├── register-app-user-tenant (depends_on: backend)
                                    └── nginx (depends_on: backend)
```

### Root compose.yaml Pattern

```yaml
services:
  backend-service:
    image: "eclipse-temurin:${JAVA_VERSION}"
    working_dir: /app
    env_file:
      - ./docker/backend-service/env/app.env
      - ./docker/backend-service/onboarding/env/${AUTH_MODE}.env
    volumes:
      - ./backend/build/distributions/backend.tar:/backend.tar
      - ./docker/backend-service/start.sh:/app/start.sh
      - onboarding:/onboarding
      - ./backend/build/otel-agent/opentelemetry-javaagent-${OTEL_AGENT_VERSION}.jar:/otel-agent.jar
    command: /app/start.sh
    ports:
      - "${BACKEND_PORT}:${BACKEND_PORT}"
    depends_on:
      pqs-app-provider:
        condition: service_started
      splice-onboarding:
        condition: service_healthy

  register-app-user-tenant:
    extends:
      file: ${MODULES_DIR}/splice-onboarding/compose.yaml
      service: splice-onboarding
    volumes:
      - ./docker/register-app-user-tenant/${AUTH_MODE}.sh:/app/scripts/on/register-app-user-tenant.sh
    entrypoint: ["/entrypoint.sh", "--exit-on-finish"]
    depends_on:
      backend-service:
        condition: service_started

  nginx:
    volumes:
      - ./frontend/dist/:/usr/share/nginx/html
      - ./config/nginx/frontend.conf:/etc/nginx/templates/frontend.conf.template
    depends_on:
      backend-service:
        condition: service_started

  splice-onboarding:
    volumes:
      - ./docker/backend-service/onboarding/onboarding.sh:/app/scripts/on/backend-service.sh
      - ./daml/my-contracts/.daml/dist/my-app-contracts-0.0.1.dar:/canton/dars/my-app-contracts-0.0.1.dar
```

### Shared Volume Pattern

The `onboarding` volume is the communication channel between init containers and the backend:

```
splice-onboarding container:
  1. Discovers party IDs from Canton participants
  2. Creates Canton user for backend
  3. Writes credentials to /onboarding/backend-service/on/backend-service.sh

backend-service container:
  1. Sources /onboarding/backend-service/on/backend-service.sh
  2. Gets APP_PROVIDER_PARTY, LEDGER_TOKEN, etc.
  3. Starts Spring Boot with resolved environment
```

---

## 13. Authentication Architecture

### Two Separate User Systems

**Canton Users** (infrastructure level):
- Access control records on a participant node
- Created via `POST /v2/users` with `primaryParty` and rights
- Your backend authenticates to Canton as one Canton user
- Answer: "Can this API caller submit commands as this party?"

**Application Users** (your code):
- Human identities in Spring Security or Keycloak
- Mapped to parties via `TenantPropertiesRepository`
- Answer: "Which human is using the UI and what party do they represent?"

### The Translation Layer

```
Human ──── OAuth2/Keycloak ───> Backend ──── Canton JWT ───> Participant
           (app layer)                      (infra layer)
```

The backend translates between the two: authenticates humans via OAuth2 (or shared-secret), looks up their party, then uses Canton user credentials to submit commands.

### Shared-Secret Mode (Development)

- Spring Security in-memory users
- No password required (or `{noop}` password)
- Static JWT token for Canton Ledger API
- Profile: `shared-secret`

### OAuth2 Mode (Production)

- Keycloak as OIDC identity provider
- JWT validation from issuer
- Multi-tenant client registration
- CSRF protection via cookies
- Profile: `oauth2`

### Security Interfaces (Reusable)

```java
public interface AuthenticatedPartyProvider {
    Optional<String> getParty();   // Get current user's Daml party ID
    String getPartyOrFail();
}

public interface TokenProvider {
    String getToken();   // Get Canton Ledger API bearer token
}

public interface AuthenticatedUserProvider {
    Optional<AuthenticatedUser> getUser();   // Get current user info
}
```

### AuthUtils (Reusable)

```java
public class AuthUtils {
    // Only the app provider (admin) can call this
    public <T> CompletableFuture<ResponseEntity<T>> asAdminParty(
        Function<String, CompletableFuture<ResponseEntity<T>>> fn)

    // Any authenticated party can call this
    public <T> CompletableFuture<ResponseEntity<T>> asAuthenticatedParty(
        Function<String, CompletableFuture<ResponseEntity<T>>> fn)
}
```

---

## 14. Onboarding and Bootstrap Sequence

### Five-Phase Startup

```
Phase 0: Configuration
  make setup → writes .env.local (PARTY_HINT, AUTH_MODE)

Phase 1: Infrastructure starts
  postgres → canton → splice
  (Each service waits for its dependency to be healthy)

Phase 2: Party discovery (splice-onboarding container)
  - Query Canton participants for party IDs created by Splice validators
  - Export APP_PROVIDER_PARTY, APP_USER_PARTY to shared volume
  - Clean up default Canton users

Phase 3: Backend Canton user creation (splice-onboarding runs backend script)
  - POST /v2/users → create "ledger-api-user"
  - POST /v2/users/ledger-api-user/rights → grant CanActAs + CanReadAs
  - Upload DAR files to participant

Phase 4: Application starts
  - backend-service reads party + credentials from shared volume
  - Starts Spring Boot

Phase 5: Tenant registration (register-app-user-tenant container)
  - POST /admin/tenant-registrations to backend API
  - Registers app-user tenant with party ID and wallet URL
```

### Onboarding Utility Functions (utils.sh)

```bash
get_admin_token()     # Get OAuth2 client credentials token
create_user()         # Create Canton user via JSON API
grant_rights()        # Grant CanActAs/CanReadAs rights
upload_dars()         # Upload all .dar files from /canton/dars/
curl_check()          # HTTP request with error handling
```

---

## 15. PQS (Projected Query Service) Integration

PQS (implemented by "Scribe") continuously streams committed transactions from Canton and projects active contract state into PostgreSQL tables.

### How PQS Works

```
Canton Participant ──streams transactions──> PQS/Scribe ──projects──> PostgreSQL
                                                                         │
Your Backend ──────────JDBC/SQL queries────────────────────────────────>──┘
```

### Querying Active Contracts

PQS provides SQL functions:
```sql
-- Get all active contracts of a template type
SELECT * FROM active('MyOrg.MyApp:MyModule:MyTemplate')

-- Filter by payload fields (JSON operators)
SELECT * FROM active('MyOrg.MyApp:MyModule:MyTemplate')
WHERE payload->>'owner' = 'party_id_here'

-- JOIN across template types
SELECT parent.contract_id, parent.payload,
       child.contract_id, child.payload
FROM active('MyOrg.MyApp:MyModule:Parent') parent
LEFT JOIN active('MyOrg.MyApp:MyModule:Child') child
  ON parent.payload->>'id' = child.payload->>'parentId'
```

### PQS Configuration

Each participant can have its own PQS instance:
- `pqs-app-provider` (always enabled)
- `pqs-app-user` (optional, for testing)
- `pqs-sv` (optional)

PQS needs:
- Ledger API connection (to stream from)
- PostgreSQL database (to write to)
- Canton user credentials (to authenticate)

---

## 16. Token Standard Integration

### When You Need Token Integration

If your app requires payment/fee collection via Canton Coin (CC), you integrate with the Splice Token Standard.

### Integration Pattern

1. **Daml Contract**: Implement `AllocationRequest` interface
2. **Backend**: Use `TokenStandardProxy` to get registry admin and transfer context
3. **Wallet**: Automatically detects your contract as a pending payment

### Required Data Dependencies (daml.yaml)

```yaml
data-dependencies:
  - ../dars/splice-api-token-metadata-v1-1.0.0.dar
  - ../dars/splice-api-token-holding-v1-1.0.0.dar
  - ../dars/splice-api-token-allocation-v1-1.0.0.dar
  - ../dars/splice-api-token-allocation-request-v1-1.0.0.dar
```

### Backend: TokenStandardProxy

```java
// Get the registry admin party (needed for InstrumentId)
String registryAdmin = tokenProxy.getRegistryAdminId().join();
InstrumentId instrumentId = new InstrumentId(registryAdmin, "Amulet");

// Get transfer context for completing allocation
Optional<ChoiceContext> ctx = tokenProxy.getAllocationTransferContext(allocationCid).join();
List<DisclosedContract> disclosed = ctx.map(c -> c.getDisclosedContracts()).orElse(List.of());

// Exercise the choice with disclosed contracts
ledgerApi.exerciseAndGetResult(contractId, choice, commandId, disclosed);
```

### OpenAPI Specs for Token Standard

The backend build generates Java clients from two token standard OpenAPI specs:
- `metadata-v1.yaml` → `DefaultMetadataApi` (get registry admin)
- `allocation-v1.yaml` → `DefaultAllocationApi` (get allocation context)

---

## 17. DevNet Deployment

### Architecture Difference: LocalNet vs DevNet

**LocalNet** (development):
- All components run locally in Docker (Canton + Splice + your app)
- Single machine, shared PostgreSQL

**DevNet** (staging/testing):
- Canton + Splice run on an external validator node (separate docker-compose in `splice-node` repo)
- Your app connects to the external validator via network
- Requires VPN access to Canton DevNet
- OAuth2 is always required (no shared-secret mode)

```
LocalNet:                        DevNet:
┌──────────────────────┐         ┌──────────────────────┐   ┌───────────────────┐
│ Your Docker Compose  │         │ Your Docker Compose  │   │ splice-node       │
│ ┌──────┐ ┌────────┐ │         │ ┌──────┐ ┌────────┐ │   │ (separate repo)   │
│ │Canton│ │ Splice │ │         │ │  KC  │ │  PQS   │ │   │ ┌──────┐ ┌─────┐ │
│ │      │ │        │ │         │ └──────┘ └────────┘ │   │ │Canton│ │Splice│ │
│ └──────┘ └────────┘ │         │ ┌──────┐ ┌────────┐ │   │ │      │ │      │ │
│ ┌──────┐ ┌────────┐ │         │ │Backend│ │ nginx │ │   │ └──────┘ └─────┘ │
│ │ PQS  │ │Keycloak│ │         │ └──────┘ └────────┘ │   └───────────────────┘
│ └──────┘ └────────┘ │         └──────────┬───────────┘          ▲
│ ┌──────┐ ┌────────┐ │                    │                      │
│ │Backend│ │ nginx │ │                    └──────connects to─────┘
│ └──────┘ └────────┘ │
└──────────────────────┘
```

### DevNet Configuration Differences

| Setting | LocalNet | DevNet |
|---------|----------|--------|
| Ledger host | `canton:3901` | `splice-validator-nginx-1:80` |
| Backend port | 8080 | 8089 (avoids conflict) |
| Auth mode | Configurable | Always OAuth2 |
| Docker network | `quickstart` | `quickstart-devnet` |
| Canton/Splice | Local containers | External splice-node |

### DevNet Deployment Steps

1. Get VPN credentials from Canton Foundation
2. Clone and start `splice-node` validator (separate repo)
3. Configure host entries (`json-ledger-api.localhost`, `validator.localhost`, etc.)
4. Run your app's DevNet module: `./docker/modules/devnet/start.sh`
5. Upload your DAR files to the external validator
6. Start your frontend with Vite dev server

### DevNet Module Structure

```
docker/modules/devnet/
├── compose.yaml      # Your services + Keycloak + PQS (no Canton/Splice)
├── compose.env       # DevNet-specific versions and connection strings
├── start.sh          # Startup script with prerequisite checks
├── stop.sh           # Shutdown script
├── env/
│   ├── common.env    # Connection to external validator
│   ├── backend.env   # Backend config for DevNet
│   └── pqs.env       # PQS connection to external participant
├── onboarding/       # Bootstrap scripts for DevNet
│   ├── backend-onboarding.sh
│   └── utils.sh
└── conf/             # Keycloak realm config for DevNet
```

---

## 18. Environment Configuration

### Configuration Hierarchy

```
.env                    # Checked in: versions, ports, network name
.env.local              # Generated: party hint, auth mode, feature flags
docker/modules/*/env/   # Per-module env files (infra-specific)
docker/backend-service/env/  # Backend-specific env vars
```

### .env (Base - checked into repo)

```bash
DAML_RUNTIME_VERSION=3.4.10
SPLICE_VERSION=0.5.3
DOCKER_NETWORK=my-app-name
JAVA_VERSION=21-jdk
BACKEND_PORT=8080
OTEL_AGENT_VERSION=2.10.0
SCRIBE_VERSION=0.6.15
SPLICE_APP_UI_NETWORK_NAME="Canton Network"
SPLICE_APP_UI_AMULET_NAME="Canton Coin"
SPLICE_APP_UI_AMULET_NAME_ACRONYM="CC"
```

### .env.local (Generated by `make setup`)

```bash
PARTY_HINT=my-company-1          # Unique identifier for your deployment
AUTH_MODE=shared-secret           # or oauth2
OBSERVABILITY_ENABLED=false       # true to enable Grafana/Prometheus/etc.
TEST_MODE=off                     # on for integration tests
```

### Backend app.env

```bash
BACKEND_PORT=8080
LEDGER_HOST=canton
LEDGER_PORT=3901
POSTGRES_HOST=postgres
POSTGRES_DATABASE=pqs-app-provider
POSTGRES_USERNAME=cnadmin
POSTGRES_PASSWORD=cnadmin
REGISTRY_BASE_URI=http://splice:5012
SPRING_PROFILES_ACTIVE=${AUTH_MODE}
```

---

## 19. Port Conventions

### Application Ports

| Service | Port | Purpose |
|---------|------|---------|
| App User UI | 2000 | Frontend for app users |
| App Provider UI | 3000 | Frontend for app provider (admin) |
| Super Validator UI | 4000 | SV management |
| Backend Service | 8080 | REST API |
| Swagger UI | 9090 | API documentation |

### Canton Ports (Pattern: Prefix + Suffix)

| Participant | Prefix | Ledger API | Admin API | JSON API | Health |
|------------|--------|-----------|-----------|----------|--------|
| app-user | 2 | 2901 | 2902 | 2975 | 2961 |
| app-provider | 3 | 3901 | 3902 | 3975 | 3961 |
| sv | 4 | 4901 | 4902 | 4975 | 4961 |

### Splice Ports

| Validator | Port |
|-----------|------|
| app-user | 2903 |
| app-provider | 3903 |
| sv | 4903 |

### Infrastructure Ports

| Service | Port |
|---------|------|
| PostgreSQL | 5432 |
| Keycloak | 8082 |
| Grafana | 3030 |

---

## 20. Reusable Code Patterns

These files should be copied (with package renaming) into any new Canton Network project:

### Backend (Highly Reusable)

| File | Purpose | Customization Needed |
|------|---------|---------------------|
| `LedgerApi.java` | gRPC Ledger API client | None (fully generic) |
| `Pqs.java` | PQS JDBC client | None (fully generic) |
| `TokenStandardProxy.java` | Token registry HTTP client | None |
| `Auth.java` | Auth mode enum | None |
| `AuthUtils.java` | Auth helper methods | None |
| `AuthenticatedPartyProvider.java` | Interface | None |
| `TokenProvider.java` | Interface | None |
| `SharedSecretConfig.java` | Dev auth config | Minor (user list) |
| `OAuth2Config.java` | Prod auth config | Minor (roles/claims) |
| `TenantPropertiesRepository.java` | Multi-tenant registry | None |
| `LedgerConfig.java` | Ledger connection config | None |
| `AdminApiImpl.java` | Tenant CRUD | None |
| `UserApiImpl.java` | Current user info | None |
| `FeatureFlagsApiImpl.java` | Auth mode flags | None |

### Frontend (Reusable)

| File | Purpose | Customization Needed |
|------|---------|---------------------|
| `api.ts` | OpenAPI client | None |
| `userStore.tsx` | Auth state management | None |
| `toastStore.tsx` | Notification system | None |
| `LoginView.tsx` | Auth-mode-aware login | Minor (branding) |
| `Header.tsx` | Nav with user/wallet | Minor (links) |
| `commandId.ts` | Unique ID generation | None |
| `error.ts` | Error handling HOC | None |

### Infrastructure (Copy as-is)

| Directory | Purpose |
|-----------|---------|
| `docker/modules/localnet/` | Full local Canton + Splice stack |
| `docker/modules/splice-onboarding/` | Party discovery + user creation |
| `docker/modules/pqs/` | PQS streaming containers |
| `docker/modules/keycloak/` | OAuth2 identity provider |
| `docker/modules/observability/` | Monitoring stack |
| `docker/modules/devnet/` | DevNet deployment config |

---

## 21. Generating a New Project Checklist

When generating a new Canton Network application, follow these steps:

### Step 1: Define Your Domain

- [ ] Name your application (e.g., `my-canton-marketplace`)
- [ ] Define your Daml contract templates and choices
- [ ] Decide if you need token standard integration (payments)
- [ ] Define your API endpoints in an OpenAPI spec
- [ ] Identify your parties (who are the actors?)

### Step 2: Scaffold the Project

- [ ] Create directory structure (see [Section 3](#3-project-directory-structure))
- [ ] Copy infrastructure modules from quickstart (`docker/modules/`)
- [ ] Set up Gradle multi-module build (root + daml + backend)
- [ ] Set up frontend with Vite + React + TypeScript
- [ ] Create `.env` with your versions and project name
- [ ] Create `Makefile` with standard targets

### Step 3: Write Daml Contracts

- [ ] Create `daml.yaml` with SDK version and data-dependencies
- [ ] Write template files in `daml/my-contracts/daml/`
- [ ] Implement `AllocationRequest` interface if you need payments
- [ ] Write Daml Script tests in separate test package

### Step 4: Create OpenAPI Specification

- [ ] Define all REST endpoints in `common/openapi.yaml`
- [ ] Include standard endpoints: feature-flags, auth, admin, user
- [ ] Add your domain-specific endpoints
- [ ] Use colon-suffix convention for actions: `/{id}:my-action`
- [ ] Include `commandId` header parameter on all write endpoints

### Step 5: Build the Backend

- [ ] Copy reusable classes (LedgerApi, Pqs, TokenStandardProxy, Auth*, etc.)
- [ ] Configure `build.gradle.kts` with all code generation plugins
- [ ] Implement service classes for your OpenAPI endpoints
- [ ] Create `DamlRepository` with your domain-specific PQS queries
- [ ] Configure `application.yml` with tenant and ledger settings

### Step 6: Build the Frontend

- [ ] Set up OpenAPI client generation
- [ ] Create React Context stores for your domain entities
- [ ] Build views for each user workflow
- [ ] Create login flow (auth-mode aware)
- [ ] Add wallet link in header (for token integration)

### Step 7: Configure Docker

- [ ] Create root `compose.yaml` with backend, nginx, register-tenant
- [ ] Mount your DAR file into splice-onboarding volume
- [ ] Configure nginx reverse proxy
- [ ] Set up onboarding scripts for your tenants
- [ ] Create per-auth-mode env files

### Step 8: Test and Deploy

- [ ] Run `make setup` for local configuration
- [ ] Run `make build && make start` for local development
- [ ] Write Playwright E2E tests
- [ ] Configure DevNet module for staging deployment
- [ ] Upload DAR to DevNet validator

### Key Decisions to Make

| Decision | Options | Default |
|----------|---------|---------|
| Auth mode | shared-secret (dev) / oauth2 (prod) | shared-secret |
| Token integration | Yes (payments) / No (simple contracts) | Depends on use case |
| Number of participants | 2 (provider + user) / 3+ (multi-org) | 2 |
| Observability | Enabled / Disabled | Disabled |
| Frontend framework | React (standard) | React + Vite |

---

## Quick Reference: Key Technologies and Versions

| Technology | Version | Purpose |
|-----------|---------|---------|
| Daml SDK | 3.4.10 | Smart contract language |
| Splice | 0.5.3 | Canton Network services |
| Java | 21 | Backend runtime |
| Spring Boot | 3.4.2 | Backend framework |
| React | 18.3.1 | Frontend framework |
| TypeScript | 5.x | Frontend language |
| Vite | 6.4.1 | Frontend build tool |
| PostgreSQL | 14 | Database |
| Scribe (PQS) | 0.6.15 | Ledger-to-SQL projection |
| Keycloak | 26.1.0 | OAuth2/OIDC provider |
| OpenTelemetry Agent | 2.10.0 | Observability |
| Gradle | 8.x | Build tool |
| Docker Compose | 2.x | Container orchestration |
