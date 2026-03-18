# CLAUDE.md - Canton Network Quickstart

## Project Overview

Canton Network (CN) Quickstart - a scaffolding project for building Canton Network applications. Implements a licensing workflow (app install requests, license creation, renewal, expiry) with Splice token standard integration.

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Smart Contracts | Daml | 3.4.10 |
| Backend | Spring Boot (Java) | 3.4.2 / Java 21 |
| Frontend | React + TypeScript + Vite | 18.3.1 / 5.5.3 / 6.4.1 |
| Infrastructure | Docker Compose + Splice LocalNet | 0.5.3 |
| Auth | OAuth2 (Keycloak) or Shared Secret | - |
| API Spec | OpenAPI 3.0 | - |
| E2E Tests | Playwright | - |
| Build | Gradle (backend/daml) + npm (frontend) | - |

## Directory Structure

All application code lives under `quickstart/`:

```
quickstart/
├── daml/licensing/daml/Licensing/   # Daml smart contracts
│   ├── License.daml                 # License template (renew, expire)
│   ├── AppInstall.daml              # AppInstall + AppInstallRequest templates
│   └── Util.daml                    # require() helper
├── daml/licensing-tests/            # Daml Script tests
├── backend/src/main/java/com/digitalasset/quickstart/
│   ├── App.java                     # Spring Boot entry point
│   ├── config/                      # LedgerConfig, PostgresConfig, SecurityConfig
│   ├── service/                     # API implementations (License, AppInstall, Admin, etc.)
│   ├── security/                    # Auth, OAuth2, SharedSecret
│   ├── ledger/                      # LedgerApi, TokenStandardProxy
│   ├── pqs/                         # Projected Query Service client
│   ├── repository/                  # DamlRepository, TenantPropertiesRepository
│   └── utility/                     # ObjectMapper, Tracing, Utils
├── frontend/src/
│   ├── views/                       # HomeView, LoginView, AppInstallsView, LicensesView
│   ├── components/                  # Header, Modal, Toast, DurationInput
│   ├── stores/                      # React Context stores (user, license, appInstall, toast)
│   ├── utils/                       # format, duration, commandId, error
│   ├── api.ts                       # Axios + OpenAPI client
│   └── types.ts                     # TypeScript type definitions
├── common/openapi.yaml              # Shared OpenAPI spec (generates backend controllers + frontend types)
├── docker/modules/                  # Docker Compose modules (localnet, keycloak, pqs, observability)
├── compose.yaml                     # Main Docker Compose orchestration
├── Makefile                         # Build orchestration
├── build.gradle.kts                 # Gradle root config
└── .env                             # Environment variables
```

## Build & Run Commands

All commands run from `quickstart/`:

```bash
# First-time setup (interactive profile configuration)
make setup

# Build everything (frontend + backend + daml + docker images)
make build

# Start the full stack
make start

# Stop everything
make stop

# Restart after changes
make restart
```

### Individual builds

```bash
make build-frontend          # npm install && npm run build
make build-backend           # ./gradlew :backend:build
make build-daml              # ./gradlew :daml:build distTar
make build-docker-images     # docker compose build
```

### Rebuild after code changes

```bash
make restart-backend         # Rebuild + restart backend container
make restart-frontend        # Rebuild + restart nginx container
make start-vite-dev          # Start with Vite hot-reload for frontend dev
```

### Testing

```bash
make test                    # Run Daml unit tests
make test-daml               # Run Daml Script tests specifically
make integration-test        # Run Playwright E2E tests (requires oauth2 + test mode)
```

### Daml SDK

```bash
make install-daml-sdk        # Install the Daml SDK
make check-daml-sdk          # Verify SDK version
make shell                   # Interactive Daml shell (Docker)
make canton-console          # Canton console (Docker)
```

### Cleanup

```bash
make clean                   # Clean build artifacts
make clean-docker            # Stop + remove Docker containers/volumes
make clean-all               # Clean everything
```

## Key Ports

| Service | Port |
|---------|------|
| App User UI | 2000 |
| App Provider UI | 3000 |
| Super Validator UI | 4000 |
| Backend Service | 8080 |
| Swagger UI | 9090 |
| App User Ledger API | 2901 |
| App Provider Ledger API | 3901 |
| Super Validator Ledger API | 4901 |
| PostgreSQL | 5432 |
| Remote Debug | 5005 |

## Daml Contract Model

The licensing workflow has these templates:

- **AppInstallRequest** - User requests to install an app (signatory: user, observer: provider)
- **AppInstall** - Active installation (signatory: provider + user). Choices: `AppInstall_CreateLicense`, `AppInstall_Cancel`
- **License** - Active license between provider and user. Choices: `License_Renew` (with token fee payment), `License_Expire`
- **LicenseRenewalRequest** - Pending renewal with fee settlement via Splice token allocation API

Daml config: `quickstart/daml/licensing/daml.yaml` (SDK 3.4.10, package: quickstart-licensing v0.0.1)

Data dependencies: Splice token APIs (MetadataV1, HoldingV1, AllocationV1, AllocationRequestV1)

## API

The OpenAPI spec at `quickstart/common/openapi.yaml` defines all endpoints. Tags: Feature Flags, Auth, Admin, App Installs, App Install Requests, Licenses, License Renewal Requests.

Generated artifacts:
- Backend: Spring controllers generated via OpenAPI Gradle plugin
- Frontend: TypeScript types via `npm run gen:openapi`

## Configuration

- `.env` - Base environment variables (versions, ports, network names)
- `.env.local` - Local overrides (generated by `make setup`): AUTH_MODE, PARTY_HINT, OBSERVABILITY_ENABLED, TEST_MODE
- `AUTH_MODE=shared-secret` (default) or `AUTH_MODE=oauth2` (adds Keycloak)

## Code Conventions

- **Daml**: Templates/Choices in UpperCamelCase, fields in lowerCamelCase, choice names prefixed with template name (e.g., `License_Renew`)
- **Java Backend**: Package `com.digitalasset.quickstart`, service classes implement OpenAPI-generated interfaces (e.g., `LicenseApiImpl`)
- **Frontend**: React Context pattern for state management, views in `views/`, reusable UI in `components/`, business logic in `stores/`
