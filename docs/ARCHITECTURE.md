# System Architecture Specification (Architecture.md)

**Document Version:** `2.0.0`  
**Status:** Approved Specification  
**Architecture Style:** Domain-Driven Modular Monolith  
**Language/Runtime:** Go (Golang 1.22+)  
**Primary Databases:** PostgreSQL (Multi-tenant DB-per-Org via Raw Queries/`pgx`), BadgerDB (LSM Engine), Rust Redis (RESP2 RESP Token/Cache Store)

---

## 1. Architectural Strategy & High-Level Overview

The system is structured as a **Domain-Driven Modular Monolith**. This pattern provides rapid development velocity, unified deployments, and simple context boundary management while strictly isolating internal domain packages. If high-throughput components (e.g., Log Telemetry or API Docs) require independent scaling in the future, they can be extracted into dedicated microservices with zero architectural refactoring of business logic.

```
+-----------------------------------------------------------------------------------+
|                                 HTTP Gateway Router                               |
|                     (Chi / Fiber Router + Middleware Chain)                       |
+-----------------------------------------------------------------------------------+
                                          │
        +---------------------------------+----------------------------------+
        │                                 │                                  │
        ▼                                 ▼                                  ▼
+------------------------+    +------------------------+    +------------------------+
|    Tenant Module       |    |   Telemetry Module     |    |    API Docs Module     |
| (Auth, BYOK, Slugs)    |    | (AOF Batch Ingestion)  |    | (SHA-256 Content-Addr) |
+------------------------+    +------------------------+    +------------------------+
        │                                 │                                  │
        ▼                                 ▼                                  ▼
+------------------------+    +------------------------+    +------------------------+
| PostgreSQL (DB A / B)  |    |  BadgerDB (LSM KV)     |    | PostgreSQL + BadgerDB  |
+------------------------+    +------------------------+    +------------------------+
```

### 1.1 Refined Architectural Guarantees
1. **Collision-Resistant Workspace Routing:** Slugs are generated using `slugify(name) + "-" + crypto_rand(5)` (e.g., `acme-corp-7x8q2`) to guarantee absolute global slug uniqueness across tenants.
2. **Dynamic Per-Project API Tables:** When a project enables API Documentation, the tenant service executes dynamic DDL to provision an isolated `project_<uuid>_apis` SQL table directly inside DB B.
3. **Content-Addressable API Spec Versioning:** API schema payloads are SHA-256 hashed prior to BadgerDB LSM writes. Code reverts (V2 $\rightarrow$ V1) are detected automatically via hash matching, pointing head pointers back without duplicating version nodes.
4. **Non-Blocking Telemetry & Client AOF Buffer:** Client SDKs write telemetry logs to a local Append-Only File (AOF) or ring buffer on the host microservice, streaming batch payload requests (`POST /api/v1/telemetry/batch`) asynchronously to prevent network latency overhead.
5. **BYOK Security Perimeter:** Encryption keys for external LLMs/AI models are isolated inside `tenant_ai_keys` within DB B and decrypted in-memory only during tool execution.

---

## 2. Directory & Package Structure

The codebase follows the standard Go project layout with strict encapsulation rules. Packages under `/core` must never depend on packages under `/modules`.

```
.
├── cmd/
│   └── server/
│       └── main.go                 # Entry point: dependency wiring, route registration, graceful shutdown
├── core/                           # Shared infrastructure & platform primitives
│   ├── config/                     # Environment variable loading & validation
│   ├── database/                   # Global DB A pool & Dynamic LRU Tenant Pool Registry (DB B)
│   ├── errors/                     # Standardized domain error codes and responses
│   ├── logger/                     # Structured logging (slog / zap)
│   ├── middleware/                 # Shared HTTP middleware (Workspace resolution, JWT, Recovery)
│   ├── redis/                      # RESP2 client integration with Rust Redis token store
│   └── router/                     # Central router initialization and interface contracts
├── models/                         # Shared entity models & AuditMixin structs
├── modules/                        # Domain feature modules (Modular Monolith boundaries)
│   ├── apidocs/                    # API Spec management, dynamic table execution & SHA-256 versioning
│   │   ├── controller.go           # HTTP payload parsing & HTTP response formatting
│   │   ├── enums.go                # API doc specific constants and status enums
│   │   ├── repository.go           # Raw SQL & BadgerDB query implementation
│   │   ├── routes.go               # Endpoint mapping
│   │   └── service.go              # SHA-256 hash calculation, revert detection & version chaining
│   ├── mcp/                        # Model Context Protocol server tools & BYOK key executor
│   ├── projects/                   # Project CRUD & dynamic feature table provisioning
│   ├── telemetry/                  # AOF batch ingestion pipeline into BadgerDB
│   └── tenant/                     # Workspace slug generation & local tenant authentication
│       ├── controller.go
│       ├── enums.go
│       ├── repository.go
│       ├── routes.go
│       └── service.go
├── migrations/                     # Raw SQL migration files managed via Goose
│   ├── global/                     # DB A migration files (Organizations, Features)
│   └── tenant/                     # Tenant DB B migration templates (Users, BYOK, Projects)
├── go.mod
└── go.sum
```

---

## 3. Module Anatomy & Layering Rules

Each module within `/modules` enforces strict internal tiering:

1. **Routes (`routes.go`):** Registers endpoints with the HTTP engine and attaches module-specific middleware.
2. **Controller (`controller.go`):** Parses HTTP requests, validates incoming payloads (DTOs), delegates to the Service, and formats JSON/HTTP responses.
3. **Service (`service.go`):** Implements core business logic, computes SHA-256 payload hashes, orchestrates dynamic project DDL execution, enforces authorization policies, and manages cache interactions.
4. **Repository (`repository.go`):** Handles all database interactions using **raw SQL queries with `pgx`** for PostgreSQL, or direct byte manipulation for BadgerDB. Direct SQL queries outside of `repository.go` are strictly forbidden.

---

## 4. Gang of Four (GoF) Design Patterns Mapping

| GoF Pattern | Architectural Location | Implementation & Purpose |
| :--- | :--- | :--- |
| **Singleton** | `core/config`, `core/logger`, DB A Pool | Ensured thread-safely via `sync.Once` to maintain single runtime instances of global infrastructure pools. |
| **Factory Method** | `core/database/registry.go` | The LRU Tenant Pool Registry acts as a factory, dynamically instantiating, configuring, and caching `pgxpool.Pool` connections for tenant DB B instances based on collision-resistant `workspace_url`. |
| **Chain of Responsibility** | `core/middleware` | HTTP requests pass through sequential middleware steps (Recovery $\rightarrow$ Workspace Resolution $\rightarrow$ Authentication $\rightarrow$ Authorization $\rightarrow$ Handler). Failure at any node halts execution immediately. |
| **Strategy** | Authentication & Ingestion | Enables runtime switching of auth mechanisms (Workspace Email/Password vs. Scoped API Key) and ingestion destinations (BadgerDB KV store vs. Relational metadata). |
| **Proxy** | `core/redis` Token Caching | Intercepts session/JWT verification requests, querying the custom Rust Redis cache before falling back to tenant database queries. |
| **Facade** | Module Services & Router | Exposes a clean, unified API interface to HTTP controllers while encapsulating internal database access, cache invalidation, and version control complexity. |

---

## 5. End-to-End Request Lifecycle & Multi-Tenant Routing

```
[ Client Request ]
       │  (HTTP Request with Workspace Slug / URL Header)
       ▼
┌────────────────────────────────────────────────────────┐
│ 1. HTTP Gateway (Router)                               │
└───────────────────────┬────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────────┐
│ 2. Workspace Resolution Middleware                     │
│    - Extracts collision-resistant workspace_url        │
│    - Queries DB A (or Redis Cache) for DB B creds      │
│    - Requests pgxpool.Pool from LRU Tenant Registry    │
│    - Injects tenant DB pool pointer into r.Context()   │
└───────────────────────┬────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────────┐
│ 3. Authentication & RBAC Middleware                    │
│    - Validates JWT against Rust Redis token store      │
│    - Verifies user permissions within tenant scope     │
└───────────────────────┬────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────────┐
│ 4. Module Controller                                   │
│    - Binds JSON request body to DTO                    │
│    - Executes payload structural validation            │
└───────────────────────┬────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────────┐
│ 5. Module Service                                      │
│    - Executes domain logic & SHA-256 payload hashing   │
│    - Provisions dynamic project tables if required     │
│    - Applies AuditMixin attributes (created_by, etc.)  │
└───────────────────────┬────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────────┐
│ 6. Module Repository                                   │
│    - Extracts tenant DB pool from r.Context()          │
│    - Executes raw SQL / BadgerDB byte operations       │
└───────────────────────┬────────────────────────────────┘
                        │
                        ▼
[ HTTP 200 / 201 Response returned to Client ]
```

---

## 6. Database Migrations & Data Access

1. **Raw Queries & High Performance:** PostgreSQL access avoids heavy ORMs. All SQL interaction is written in pure SQL using `pgx/v5` for maximum query optimization and zero reflection overhead.
2. **Schema Migration Strategy (`goose`):**
   * **Global Schema (DB A):** Migrations stored in `/migrations/global` and executed during application bootstrapping.
   * **Tenant Schema (DB B):** Migrations stored in `/migrations/tenant` (`YYYYMMDDHHMMSS_name.sql`) and executed programmatically via `goose.Up()` upon provisioning a new organization or executing automated tenant rollouts.
3. **Caching Layer:** Redis operations (via custom Rust Redis RESP2 server) handle:
   * Active user sessions and invalidated JWT blacklists.
   * Workspace URL to DB B connection string resolution maps.

---
