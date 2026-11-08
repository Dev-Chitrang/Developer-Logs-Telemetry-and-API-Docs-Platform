# Stage-Wise Implementation Plan (Plan.md)

**Document Version:** `1.0.0`

**Status:** Approved Specification

**Approach:** Developer-centric refactored workflow (Skeleton $\rightarrow$ Database Schemas $\rightarrow$ Core Primitives $\rightarrow$ Feature API Modules)

## Milestone 1: Folder Structure & Package Skeleton Setup

Initialize the project directory layout, configuration files, and empty Go packages following the modular monolith architecture.

* [ ] **1.1 Workspace Initialization**
  * [ ] Initialize Go module (`go mod init controlplane`).
  * [ ] Setup `.gitignore`, `Makefile`, and `env.example`.
* [ ] **1.2 Directory Creation**
  * [ ] Create `/cmd/server` (application entry point).
  * [ ] Create `/core` infrastructure sub-directories: `/config`, `/database`, `/errors`, `/logger`, `/middleware`, `/redis`, `/router`.
  * [ ] Create `/models` shared data model package.
  * [ ] Create `/modules` domain feature directories: `/tenant`, `/projects`, `/apidocs`, `/telemetry`, `/mcp`.
  * [ ] Create `/migrations` directory structure (`/migrations/global` and `/migrations/tenant`).

## Milestone 2: Database Schemas & Migration Files (`goose`)

Create raw SQL migration files (`.sql`) for both Global DB A and Tenant DB B using `goose` migration syntax.

* [ ] **2.1 Global DB A Migrations (`/migrations/global`)**
  * [ ] `00001_create_global_organizations.sql` (UUIDv7 `id`, `name`, `workspace_url`, encrypted `db_connection_credentials`, AuditMixin fields).
  * [ ] `00002_create_global_features.sql` (UUIDv7 `id`, `name`, AuditMixin fields).
  * [ ] `00003_create_global_organization_features.sql` (UUIDv7 `id`, `organization_id`, `feature_id`, AuditMixin fields).
* [ ] **2.2 Tenant DB B Migrations (`/migrations/tenant`)**
  * [ ] `00001_create_tenant_users.sql` (UUIDv7 `id`, `name`, `email`, `password_hash`, AuditMixin fields).
  * [ ] `00002_create_tenant_projects.sql` (UUIDv7 `id`, `name`, AuditMixin fields).
  * [ ] `00003_create_tenant_user_project_mapping.sql` (UUIDv7 `id`, `user_id`, `project_id`, AuditMixin fields).
  * [ ] `00004_create_tenant_project_feature_mapping.sql` (UUIDv7 `id`, `project_id`, `feature_id`, AuditMixin fields).
  * [ ] `00005_create_tenant_apis.sql` (UUIDv7 `id`, `project_id`, `endpoint`, `current_version`, AuditMixin fields).

## Milestone 3: Core Infrastructure & Global Primitives Implementation

Implement baseline infrastructure, connection management, error handlers, and generic response templates.

* [ ] **3.1 Global Configuration & Logging (`/core/config`, `/core/logger`)**
  * [ ] Implement `core/config` environment variable parser (`viper` or `godotenv`).
  * [ ] Implement `core/logger` structured logging wrapper (`slog` / `zap`) with thread-safe `sync.Once`.
* [ ] **3.2 Response & Error Contracts (`/core/errors`, `/models`)**
  * [ ] Implement standardized Go structs and response builder for Success/Error envelopes (`models/response.go`).
  * [ ] Implement standard application error interfaces and HTTP mapping (`core/errors`).
  * [ ] Create shared `AuditMixin` struct definitions for Go models.
* [ ] **3.3 Database & Cache Connections (`/core/database`, `/core/redis`)**
  * [ ] Initialize Global DB A PostgreSQL connection pool using `pgx/v5`.
  * [ ] Implement the dynamic **LRU Tenant Pool Registry** for DB B pools (`core/database/registry.go`).
  * [ ] Implement custom Rust Redis RESP2 client wrapper for token storage (`core/redis`).
  * [ ] Initialize embedded **BadgerDB** LSM-tree engine manager.
* [ ] **3.4 Central Router & Core Middleware (`/core/router`, `/core/middleware`)**
  * [ ] Initialize main HTTP router (Chi / Fiber / Gin).
  * [ ] Implement Workspace URL resolution middleware (injects target tenant DB pool into `r.Context()`).
  * [ ] Implement JWT authentication & token validation middleware via Redis.

## Milestone 4: Feature-Wise API Module Implementation

Build feature API modules adhering to the Controller $\rightarrow$ Service $\rightarrow$ Repository layering pattern.

* [ ] **4.1 Feature 1: Workspace & Organization Management (`/modules/tenant`)**
  * [ ] Implement `repository.go`: Raw SQL queries for workspace creation & DB A lookup.
  * [ ] Implement `service.go`: Dynamic database provisioning logic and migration execution for new tenants.
  * [ ] Implement `controller.go` & `routes.go`: Endpoints `POST /api/v1/workspaces` and `GET /api/v1/workspaces/{slug}/verify`.
* [ ] **4.2 Feature 2: Workspace Auth & Session Management (`/modules/tenant/auth`)**
  * [ ] Implement `repository.go`: Local tenant `users` queries in DB B.
  * [ ] Implement `service.go`: Argon2id password verification and JWT token issuance.
  * [ ] Implement `controller.go` & `routes.go`: Endpoints `POST /auth/login` and `POST /auth/refresh`.
* [ ] **4.3 Feature 3: Projects Management (`/modules/projects`)**
  * [ ] Implement `repository.go`, `service.go`, `controller.go`, `routes.go`.
  * [ ] Endpoints: `GET /api/v1/workspaces/{slug}/projects` and `POST /api/v1/workspaces/{slug}/projects`.
* [ ] **4.4 Feature 4: Hybrid API Spec Engine & Versioning (`/modules/apidocs`)**
  * [ ] Implement `repository.go`: SQL route pointer queries + BadgerDB append-only version chain writes.
  * [ ] Implement `service.go`: Version diffing and append logic (`project_id::api_id::version::timestamp`).
  * [ ] Implement `controller.go` & `routes.go`: Endpoints for registering APIs, fetching version history chains, and appending new versions.
* [ ] **4.5 Feature 5: Telemetry & Ingestion Engine (`/modules/telemetry`)**
  * [ ] Implement `repository.go`: Async batch byte writes to BadgerDB (`project_id::timestamp`).
  * [ ] Implement in-memory Go worker channels and ring buffers for ultra-low latency ingestion.
  * [ ] Implement `controller.go` & `routes.go`: High-speed `POST /api/v1/telemetry/ingest` endpoint (`202 Accepted`).
* [ ] **4.6 Feature 6: MCP AI Agent Tools (`/modules/mcp`)**
  * [ ] Implement JSON-RPC / SSE handler for Model Context Protocol.
  * [ ] Register MCP tools: `search_api_specs`, `get_schema_by_version`, `patch_openapi_spec`.
  * [ ] Validate AI coding agent integration (Claude Code / Cursor).
