# Teable Backend Architecture Analysis

> **A comprehensive guide for newcomers to understand the Teable project structure and backend architecture.**

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Project Structure](#2-project-structure)
3. [Backend Tech Stack](#3-backend-tech-stack)
4. [Core Architecture Decisions](#4-core-architecture-decisions)
5. [Related Documentation](#5-related-documentation)

---

## 1. System Overview

### Product Positioning

**Teable** is an open-source, no-code database platform that combines:
- **Spreadsheet-like UI**: Familiar interface for non-technical users
- **Database Power**: Full PostgreSQL capabilities under the hood
- **Real-time Collaboration**: Multiple users can work simultaneously
- **Million-row Performance**: Optimized for large datasets

### Core Value Proposition

| Feature | Description |
|---------|-------------|
| **Spreadsheet UI** | Intuitive grid, kanban, calendar, and form views |
| **Database Backend** | PostgreSQL for production, SQLite for development |
| **Real-time Sync** | ShareDB-powered operational transformation |
| **Formula Engine** | 48+ functions with ANTLR4 parser |
| **API-First** | RESTful OpenAPI with comprehensive endpoints |

### Deployment Models

- **Cloud (teable.ai)**: Managed SaaS offering
- **Self-hosted**: Docker-based deployment with PostgreSQL, Redis, and MinIO
- **Development**: SQLite mode for local development

### Scale Target

- Supports **millions of rows** per table
- Designed for **hundreds of concurrent users**
- **Real-time collaboration** with sub-second sync

---

## 2. Project Structure

### Monorepo Organization

```
teable/
├── apps/
│   ├── nestjs-backend/          # NestJS API server
│   └── nextjs-app/              # Next.js frontend
├── packages/
│   ├── core/                    # Shared business logic & formula engine
│   ├── db-main-prisma/          # Prisma schema & migrations
│   ├── openapi/                 # OpenAPI specifications
│   ├── sdk/                     # Public SDK
│   ├── ui-lib/                  # Shared UI components
│   ├── common-i18n/             # Internationalization
│   ├── icons/                   # Icon library
│   └── eslint-config-bases/     # Shared ESLint configs
├── plugins/                     # Plugin system
├── dockers/                     # Docker configurations
└── docs/                        # Documentation
```

### Key Entry Points

| Entry Point | Location | Purpose |
|-------------|----------|---------|
| Backend Main | `apps/nestjs-backend/src/index.ts` | NestJS application bootstrap |
| Frontend Main | `apps/nextjs-app/src/pages/_app.tsx` | Next.js application wrapper |
| App Module | `apps/nestjs-backend/src/app.module.ts` | NestJS module composition |
| Prisma Schema | `packages/db-main-prisma/prisma/template.prisma` | Database schema template |

### Package Manager & Build

- **Package Manager**: pnpm 9.13.0+
- **Node.js**: 20.0.0+
- **Build Tool**: Turbo (monorepo build orchestration)
- **Bundler**: Webpack (backend), Next.js (frontend)

---

## 3. Backend Tech Stack

### Application Framework

| Component | Technology | Version |
|-----------|------------|---------|
| **Framework** | NestJS | 10.3.5 |
| **Runtime** | Node.js | 20.x |
| **Language** | TypeScript | 5.4.3 |
| **API Style** | REST (OpenAPI) | 3.0 |

### Database Layer

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Primary DB** | PostgreSQL | Production database |
| **Dev DB** | SQLite | Local development |
| **ORM** | Prisma | 6.2.1 |
| **Query Builder** | Knex.js | Dynamic query construction |

### Infrastructure

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Caching** | Redis / SQLite | Session, job queue, performance cache |
| **Job Queue** | BullMQ | Background job processing |
| **Real-time** | ShareDB + WebSocket | Operational transformation |
| **File Storage** | S3 / MinIO / Local | Attachment storage |
| **Auth** | Passport.js | Multi-strategy authentication |

### Observability

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Logging** | Pino | Structured logging |
| **Tracing** | OpenTelemetry | Distributed tracing |
| **Error Tracking** | Sentry | Error monitoring |

---

## 4. Core Architecture Decisions

### 4.1 Module Architecture

Teable follows **NestJS modular architecture** with 44 feature modules:

```
src/
├── app.module.ts              # Root module
├── global/                    # Global providers (DB, cache, permissions)
├── features/                  # 44 feature modules
│   ├── auth/                  # Authentication & authorization
│   ├── base/                  # Database/base management
│   ├── table/                 # Table operations
│   ├── field/                 # Field/column management
│   ├── record/                # Record CRUD
│   ├── view/                  # View management
│   └── ...                    # Other features
├── db-provider/               # Database abstraction layer
├── share-db/                  # Real-time collaboration
├── cache/                     # Caching layer
└── event-emitter/             # Event system
```

### 4.2 Database Provider Pattern

The system abstracts PostgreSQL and SQLite differences through a **Provider Pattern**:

```typescript
// Factory creates appropriate provider based on database driver
const DbProvider = {
  provide: DB_PROVIDER_SYMBOL,
  useFactory: (knex: Knex) => {
    switch (getDriverName(knex)) {
      case DriverClient.Sqlite: return new SqliteProvider(knex);
      case DriverClient.Pg: return new PostgresProvider(knex);
    }
  }
};
```

**IDbProvider Interface** provides:
- Filter query builders
- Sort query builders
- Aggregation queries
- Schema management
- Column operations

### 4.3 Dynamic Field Storage

Teable uses a **hybrid approach** for storing dynamic table structures:

```
TableMeta (logical)          Physical Table (dbTableName)
├── Field A (text)     →     "fld_xxx" VARCHAR column
├── Field B (number)   →     "fld_yyy" NUMERIC column
├── Field C (formula)  →     "fld_zzz" GENERATED column (if supported)
└── Field D (link)     →     "fld_www" JSONB column
```

**Key Design Decisions:**
- **One physical table per logical table** (not EAV pattern)
- **Dynamic column creation** via SQL DDL
- **JSONB for complex types** (attachments, links, multi-select)
- **Generated columns** for simple formulas (database-calculated)
- **Application-level calculation** for complex formulas

### 4.4 Real-time Collaboration

Built on **ShareDB** with custom adapters:

```
Client WebSocket → WsGateway → ShareDB Service → Database
                      ↓
              PubSub (Redis/Memory)
                      ↓
              Other Clients
```

**Key Components:**
- **WebSocket Gateway**: Native WS with JSON streaming
- **ShareDB Adapter**: Custom database adapter for Prisma
- **Redis PubSub**: Multi-instance synchronization
- **Presence**: Real-time user awareness

### 4.5 Authentication & Authorization

**Multi-Strategy Authentication:**
```
Request → AuthGuard → [Session | AccessToken | JWT] → Validated User
```

**RBAC Permission Model:**
```
Space (workspace)
└── Collaborator (role: owner|creator|editor|commenter|viewer)
    └── Base (database)
        └── Table → View → Record
```

**Permission Resolution:**
1. Check `@Public()` decorator
2. Validate authentication
3. Resolve resource from route params
4. Check role-based permissions
5. Intersect with access token scopes (if API)

---

## 5. Related Documentation

For detailed information on specific subsystems, see:

| Document | Description |
|----------|-------------|
| [Database Architecture](./docs/ARCHITECTURE-DATABASE.md) | Schema design, query engine, formula system |
| [Real-time Architecture](./docs/ARCHITECTURE-REALTIME.md) | WebSocket, ShareDB, event system |
| [Security Architecture](./docs/ARCHITECTURE-SECURITY.md) | Auth, permissions, access control |
| [Operations Architecture](./docs/ARCHITECTURE-OPERATIONS.md) | Jobs, caching, import/export, storage |
| [Testing & Observability](./docs/ARCHITECTURE-TESTING.md) | Testing strategy, logging, tracing |

---

## Quick Reference

### Key Configuration Files

| File | Purpose |
|------|---------|
| `apps/nestjs-backend/src/configs/` | Backend configuration modules |
| `packages/db-main-prisma/prisma/template.prisma` | Database schema |
| `dockers/` | Docker compose configurations |
| `.env` files | Environment variables |

### Important Environment Variables

```bash
# Database
DATABASE_URL=postgresql://user:pass@host:5432/db

# Cache & Queue
BACKEND_CACHE_PROVIDER=redis
BACKEND_CACHE_REDIS_URI=redis://localhost:6379

# Storage
BACKEND_STORAGE_PROVIDER=minio
BACKEND_STORAGE_PUBLIC_BUCKET=public
BACKEND_STORAGE_PRIVATE_BUCKET=private

# Auth
BACKEND_JWT_SECRET=your-secret
BACKEND_SESSION_SECRET=your-session-secret

# Observability
LOG_LEVEL=info
BACKEND_SENTRY_DSN=https://...
OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
```

### Development Commands

```bash
# Install dependencies
pnpm install

# Start development (SQLite mode)
make sqlite.mode
pnpm dev

# Start with PostgreSQL
make postgres.mode
docker compose up -d postgres redis
pnpm dev

# Run tests
pnpm g:test-unit          # Unit tests
pnpm g:test-e2e           # Integration tests

# Build
pnpm g:build              # Build all packages
```

---

## Architecture Diagrams

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   Web UI    │  │   SDK/API   │  │     WebSocket Client    │ │
│  └──────┬──────┘  └──────┬──────┘  └────────────┬────────────┘ │
└─────────┼────────────────┼──────────────────────┼───────────────┘
          │                │                      │
          ▼                ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                       API Gateway                                │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    NestJS Backend                            ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐ ││
│  │  │   Auth   │  │  OpenAPI │  │ WebSocket│  │    Guards   │ ││
│  │  │  Guards  │  │Controllers│ │  Gateway │  │ Interceptors│ ││
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬──────┘ ││
│  │       └──────────────┴────────────┴───────────────┘         ││
│  └─────────────────────────────┬───────────────────────────────┘│
└────────────────────────────────┼────────────────────────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          ▼                      ▼                      ▼
┌─────────────────┐  ┌─────────────────────┐  ┌─────────────────┐
│  Service Layer  │  │   ShareDB Service   │  │   Job Queues    │
│  ┌───────────┐  │  │  ┌──────────────┐  │  │  ┌───────────┐  │
│  │   Base    │  │  │  │   Adapter    │  │  │  │  BullMQ   │  │
│  │   Table   │  │  │  │   PubSub     │  │  │  │  Workers  │  │
│  │   Field   │  │  │  │   Presence   │  │  │  │           │  │
│  │   Record  │  │  │  └──────────────┘  │  │  └───────────┘  │
│  │   View    │  │  └─────────────────────┘  └─────────────────┘
│  └───────────┘  │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Data Layer                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────┐ │
│  │  PostgreSQL │  │    Redis    │  │   S3/MinIO  │  │ SQLite │ │
│  │  (Primary)  │  │   (Cache)   │  │  (Storage)  │  │  (Dev) │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Request Flow

```
HTTP Request
    │
    ▼
┌──────────────────┐
│ RequestInfoMiddle│ ← Extract IP, User-Agent, etc.
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│    AuthGuard     │ ← Session / Token / JWT validation
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ PermissionGuard  │ ← RBAC permission check
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   Controller     │ ← Route handling
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│    Service       │ ← Business logic
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   Repository     │ ← Data access (Prisma/Knex)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│    Database      │
└──────────────────┘
```

---

*Last updated: 2024*
