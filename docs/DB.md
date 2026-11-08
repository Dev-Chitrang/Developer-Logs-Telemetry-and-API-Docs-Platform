# Database Architecture & Schema Specification (DB.md)

**Document Version:** `2.0.0`  
**Status:** Approved Specification  
**Primary Identifiers Strategy:** `UUIDv7` (Time-ordered 128-bit UUIDs)  
**Database Topology:** Multi-Tenant Database-per-Organization (PostgreSQL) + Embedded LSM Key-Value Store (BadgerDB)

---

## 1. Executive Architectural Overview

The control plane utilizes a hybrid two-tier storage topology designed for strict tenant isolation, ultra-fast write throughput, and deterministic routing.

```
                          ┌───────────────────────────┐
                          │    Workspace URL Lookup   │
                          └─────────────┬─────────────┘
                                        │
                                        ▼
                   ┌─────────────────────────────────────────┐
                   │    Global DB (DB A: Control Plane)      │
                   │   - Global Organizations                │
                   │   - Global Features Registry            │
                   └────────────────────┬────────────────────┘
                                        │ (Resolves Connection Credentials)
                                        ▼
                 ┌─────────────────────────────────────────────┐
                 │  Tenant DB (DB B: Per-Org Relational DB)    │
                 │  - Users & Passwords (Local Auth)           │
                 │  - BYOK Keys (OpenAI, Anthropic, Gemini)    │
                 │  - Tenant & Project Feature Toggles         │
                 │  - Projects & Dynamic API Tables            │
                 └──────────────────────┬──────────────────────┘
                                        │ (Appends Payload & Telemetry)
                                        ▼
                 ┌─────────────────────────────────────────────┐
                 │    BadgerDB (NoSQL LSM-Tree Key-Value)      │
                 │  - Ingested Logs (Buffered/AOF Streams)     │
                 │  - Content-Addressable (SHA-256) API Chains │
                 └─────────────────────────────────────────────┘
```

### 1.1 Key Architectural Decisions

1. **Time-Ordered Primary Keys (`UUIDv7`):** All SQL table primary keys use `UUIDv7`. This retains global uniqueness while keeping PostgreSQL B-Tree indexes fast and sequential without index fragmentation.
2. **Collision-Resistant Unique Workspace Slugs:** Slugs are generated using `slugify(org_name) + "-" + crypto_rand(5)` (e.g., `acme-corp-7x8q2`) with a unique database constraint in DB A to prevent workspace collision attacks.
3. **Workspace-URL First Routing (Zero-Leakage Auth):** Authentication happens locally inside DB B resolved via the workspace slug pointer. User credentials never exist in DB A.
4. **Bring Your Own Key (BYOK) Storage:** Encrypted provider API keys (OpenAI, Anthropic, Gemini, etc.) are stored strictly inside the tenant database (DB B) in a dedicated multi-provider table.
5. **Dynamic Per-Project API Tables:** When a project enables the API Documentation feature, the system dynamically provisions a dedicated table (`project_<uuid>_apis`) inside DB B to ensure per-project isolation.
6. **Content-Addressable Versioning & Revert Detection:** Schema version payloads stored in BadgerDB compute a SHA-256 hash. Reverts (e.g., reverting code from V2 back to V1) are recognized automatically by checking hash matches, avoiding data duplication and keeping version trees clean.
7. **Client-Side AOF / Non-Blocking Telemetry Ingestion:** SDKs buffer log streams locally using an Append-Only File (AOF) or in-memory ring buffer, flushing asynchronously via compressed batch requests (`POST /api/v1/telemetry/batch`) to eliminate network-induced latency in client microservices.
8. **Automated Schema Migration (`goose`):** Version-controlled migrations (`YYYYMMDDHHMMSS_name.sql`) execute programmatically via `goose.Up()` upon provisioning or upgrading tenant DB B pools.

---

## 2. Architecture Diagram Placeholder

<!-- INSERT_SCHEMA_DIAGRAM_HERE -->
> **Note:** Insert your architecture/schema ER diagram image above (`Go Dev Tool_2.png`).

---

## 3. Global Control Plane (DB A - PostgreSQL)

DB A contains system-wide operational metadata, global feature definitions, and encrypted database connection credentials per organization.

### 3.1 AuditMixin Base Definition
Every table inherits the following standard audit fields:

```sql
-- AuditMixin Fields
created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
deleted_at TIMESTAMP WITH TIME ZONE NULL,
is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
created_by UUID NULL,
updated_by UUID NULL,
deleted_by UUID NULL
```

### 3.2 `global_organizations`
Stores tenant profiles and encrypted connection info for their isolated DB instances.

| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Time-ordered UUIDv7 |
| `name` | `VARCHAR(255)` | `NOT NULL` | Human-readable organization name |
| `workspace_url` | `VARCHAR(120)` | `NOT NULL, UNIQUE` | Collision-resistant slug (e.g., `acme-corp-7x8q2`) |
| `db_connection_credentials` | `TEXT` | `NOT NULL` | Encrypted JSON string containing tenant DB host, port, DB name, user, and password (AES-256-GCM) |
| *AuditMixin Fields* | — | — | Included |

### 3.3 `global_features`
System-wide feature registry.

| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Time-ordered UUIDv7 |
| `name` | `VARCHAR(100)` | `NOT NULL, UNIQUE` | Feature key (e.g., `api_documentation`, `lsm_logging`, `byok_ai`) |
| *AuditMixin Fields* | — | — | Included |

---

## 4. Tenant Data Plane (DB B - PostgreSQL Per Org)

Each organization owns a distinct PostgreSQL database provisioned dynamically during signup.

### 4.1 `users`
Local authentication and profile directory for the organization.

| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Time-ordered UUIDv7 |
| `name` | `VARCHAR(255)` | `NOT NULL` | User full name |
| `email` | `VARCHAR(255)` | `NOT NULL, UNIQUE` | User email address within this organization |
| `password_hash` | `VARCHAR(255)` | `NOT NULL` | Argon2id / bcrypt password hash |
| *AuditMixin Fields* | — | — | Included |

### 4.2 `tenant_ai_keys` (BYOK Storage)
Stores encrypted API keys for external LLM/AI providers.

| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Time-ordered UUIDv7 |
| `provider` | `VARCHAR(50)` | `NOT NULL` | Provider enum (`openai`, `anthropic`, `gemini`, `custom`) |
| `key_name` | `VARCHAR(100)` | `NOT NULL` | Label (e.g., `Production-GPT4o-Key`) |
| `encrypted_api_key` | `TEXT` | `NOT NULL` | AES-256 encrypted key payload |
| `is_active` | `BOOLEAN` | `DEFAULT TRUE` | Key enablement toggle |
| *AuditMixin Fields* | — | — | Included |

### 4.3 `tenant_organization_features`
Mapping table for features enabled specifically for this organization.

| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Time-ordered UUIDv7 |
| `feature_id` | `UUID` | `NOT NULL` | References `global_features(id)` |
| `is_enabled` | `BOOLEAN` | `DEFAULT TRUE` | Enablement status |
| *AuditMixin Fields* | — | — | Included |

### 4.4 `projects`
Workspaces created within the tenant organization.

| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Time-ordered UUIDv7 |
| `name` | `VARCHAR(255)` | `NOT NULL` | Project name |
| `api_table_name` | `VARCHAR(100)` | `NULL` | Dynamically created table name (e.g., `project_018f23a4_apis`) |
| *AuditMixin Fields* | — | — | Included |

### 4.5 `user_project_mapping`
Maps user access permissions to specific projects.

| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Time-ordered UUIDv7 |
| `user_id` | `UUID` | `NOT NULL, FK` | References `users(id)` |
| `project_id` | `UUID` | `NOT NULL, FK` | References `projects(id)` |
| *AuditMixin Fields* | — | — | Included |

### 4.6 `project_feature_mapping`
Fine-grained feature toggles per project.

| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Time-ordered UUIDv7 |
| `project_id` | `UUID` | `NOT NULL, FK` | References `projects(id)` |
| `feature_id` | `UUID` | `NOT NULL` | References `global_features(id)` |
| *AuditMixin Fields* | — | — | Included |

### 4.7 Dynamic Per-Project API Tables (`project_{project_id}_apis`)
Dynamically generated when a project enables API Documentation.

```sql
CREATE TABLE project_<project_id>_apis (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid_v7(),
    endpoint VARCHAR(500) NOT NULL,
    current_version VARCHAR(50) NOT NULL,
    current_schema_hash VARCHAR(64) NOT NULL, -- SHA-256 Hash of latest payload
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE NULL,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_by UUID NULL,
    updated_by UUID NULL,
    deleted_by UUID NULL
);
```

---

## 5. High-Throughput Engine (NoSQL - BadgerDB)

BadgerDB serves as the key-value store for append-only log streams and content-addressable schema version chains.

### 5.1 Key Encoding & Content-Addressable Hashing

#### 1. Telemetry & Log Ingestion Store (AOF Batching)
* **Key Format:**  
  `logs::{project_id}::{timestamp_nanos}::{log_id}`
* **Value Format (JSON/Protobuf):**
  ```json
  {
    "severity": "ERROR",
    "message": "connection reset by peer",
    "stack_trace": "...",
    "metadata": { "host": "runner-1", "latency_ms": 14 }
  }
  ```

#### 2. Content-Addressable API Version Store (SHA-256 Hashing)
* **Key Format:**  
  `apidocs::{project_id}::{api_id}::{version_number}::{sha256_hash}`
* **Value Format (JSON):**
  ```json
  {
    "openapi_version": "3.0.3",
    "sha256_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "is_revert": false,
    "reverted_from_version": null,
    "method": "POST",
    "request_schema": { ... },
    "response_schema": { ... },
    "author_id": "018f23a4-8b91-7d12-b234-123456789abc"
  }
  ```

---
