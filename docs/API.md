# API Contract Specification (API.md)

**Document Version:** `2.0.0`  
**Status:** Approved Specification  
**Protocol:** HTTP/1.1 & HTTP/2 (TLS 1.3 Enforced)  
**Data Format:** JSON (`application/json`)  
**Base Endpoint Prefix:** `/api/v1`

---

## 1. Standard Response Envelope Format

All endpoints return a standardized, predictable JSON envelope structure regardless of success or failure.

### 1.1 Success Response Template

```json
{
  "success": true,
  "status_code": 200,
  "message": "Operation completed successfully",
  "data": {},
  "meta": {
    "timestamp": "2026-11-08T12:00:00Z",
    "pagination": {
      "limit": 20,
      "offset": 0,
      "total": 100
    }
  }
}
```

### 1.2 Error Response Template

```json
{
  "success": false,
  "status_code": 400,
  "message": "Invalid request payload provided",
  "error": {
    "code": "VALIDATION_FAILED",
    "details": [
      {
        "field": "email",
        "issue": "must be a valid email format"
      }
    ]
  },
  "meta": {
    "timestamp": "2026-11-08T12:00:00Z"
  }
}
```

---

## 2. Common HTTP Status Codes

| Code | Status Name | Usage Description |
| :--- | :--- | :--- |
| `200 OK` | Standard Success | Successful read or update operations. |
| `201 Created` | Created | Resource or dynamic SQL table successfully created. |
| `202 Accepted` | Processing Accepted | Asynchronous tasks (e.g., AOF batch log ingestion). |
| `400 Bad Request` | Client Error | Payload parsing, invalid SHA-256 hash, or validation failure. |
| `401 Unauthorized` | Unauthenticated | Missing or expired JWT access token. |
| `403 Forbidden` | Access Denied | Insufficient permissions for tenant or project scope. |
| `404 Not Found` | Not Found | Target workspace slug, project, or API spec does not exist. |
| `409 Conflict` | Duplicate Entity | Workspace URL collision or duplicate API route. |
| `429 Too Many Requests` | Rate Limited | Exceeded API rate limits on token or ingestion routes. |
| `500 Internal Error` | Server Error | Unhandled server or database error. |

---

## 3. Module Endpoint Contracts

### 3.1 Tenant & Workspace Module (`core/modules/tenant`)

#### `POST /api/v1/workspaces`
* **Responsibility:** Provision a new organization, generate a collision-resistant workspace slug (`slugify(name) + "-" + crypto_rand(5)`), allocate its dedicated PostgreSQL DB instance (DB B), run `goose` migrations, and register default admin user.
* **Auth Required:** No (Public Endpoint)
* **Request Body:**
  ```json
  {
    "organization_name": "Acme Corp",
    "admin_name": "Jane Doe",
    "admin_email": "jane@acme.com",
    "admin_password": "SecurePassword123!"
  }
  ```
* **Response (`201 Created`):**
  ```json
  {
    "success": true,
    "status_code": 201,
    "message": "Workspace successfully provisioned",
    "data": {
      "organization_id": "018f23a4-8b91-7d12-b234-123456789abc",
      "workspace_url": "acme-corp-7x8q2",
      "admin_user_id": "018f23a4-8b91-7d12-b234-987654321xyz"
    },
    "meta": { "timestamp": "2026-11-08T12:00:00Z" }
  }
  ```

#### `GET /api/v1/workspaces/{slug}/verify`
* **Responsibility:** Public collision-resistant workspace resolution check before rendering tenant login form.
* **Auth Required:** No
* **Response (`200 OK`):**
  ```json
  {
    "success": true,
    "status_code": 200,
    "message": "Workspace exists",
    "data": {
      "workspace_url": "acme-corp-7x8q2",
      "organization_name": "Acme Corp",
      "is_active": true
    },
    "meta": { "timestamp": "2026-11-08T12:00:00Z" }
  }
  ```

---

### 3.2 BYOK (Bring Your Own Key) Management Module (`core/modules/tenant/byok`)

#### `POST /api/v1/workspaces/{slug}/byok`
* **Responsibility:** Securely add or update an encrypted AI provider API key inside DB B.
* **Auth Required:** Yes (`Bearer <access_token>`)
* **Request Body:**
  ```json
  {
    "provider": "openai",
    "key_name": "Production-GPT4o-Key",
    "api_key": "sk-proj-1234567890abcdef"
  }
  ```
* **Response (`201 Created`):**
  ```json
  {
    "success": true,
    "status_code": 201,
    "message": "BYOK API key stored securely",
    "data": {
      "key_id": "018f23a4-8b91-7d12-b234-555555555555",
      "provider": "openai",
      "key_name": "Production-GPT4o-Key",
      "is_active": true
    },
    "meta": { "timestamp": "2026-11-08T12:00:00Z" }
  }
  ```

---

### 3.3 Projects Module (`core/modules/projects`)

#### `POST /api/v1/workspaces/{slug}/projects`
* **Responsibility:** Create a project inside DB B. If `enable_api_docs` is `true`, dynamically executes DDL to provision dynamic SQL table `project_<project_id>_apis`.
* **Auth Required:** Yes (`Bearer <access_token>`)
* **Request Body:**
  ```json
  {
    "name": "Order Management Microservice",
    "enable_api_docs": true
  }
  ```
* **Response (`201 Created`):**
  ```json
  {
    "success": true,
    "status_code": 201,
    "message": "Project created with dynamic API table",
    "data": {
      "id": "018f23a4-8b91-7d12-b234-222222222222",
      "name": "Order Management Microservice",
      "api_table_name": "project_018f23a4_8b91_7d12_b234_222222222222_apis"
    },
    "meta": { "timestamp": "2026-11-08T12:00:00Z" }
  }
  ```

---

### 3.4 API Specs & SHA-256 Version Chain Module (`core/modules/apidocs`)

#### `POST /api/v1/workspaces/{slug}/projects/{project_id}/apis`
* **Responsibility:** Register a new API endpoint route pointer into the project's dynamic table (`project_<project_id>_apis`).
* **Auth Required:** Yes
* **Request Body:**
  ```json
  {
    "endpoint": "/api/v1/checkout",
    "initial_version": "v1.0.0"
  }
  ```
* **Response (`201 Created`):**
  ```json
  {
    "success": true,
    "status_code": 201,
    "message": "API endpoint pointer registered in project table",
    "data": {
      "api_id": "018f23a4-8b91-7d12-b234-333333333333",
      "endpoint": "/api/v1/checkout",
      "current_version": "v1.0.0"
    },
    "meta": { "timestamp": "2026-11-08T12:00:00Z" }
  }
  ```

#### `POST /api/v1/workspaces/{slug}/projects/{project_id}/apis/{api_id}/versions`
* **Responsibility:** Compute SHA-256 hash of payload. Check BadgerDB LSM tree for matching hash. If hash matches older version, mark as **revert** and update pointer; otherwise append new version node.
* **Auth Required:** Yes
* **Request Body:**
  ```json
  {
    "version": "v1.1.0",
    "schema_payload": {
      "method": "POST",
      "request_body": {
        "type": "object",
        "properties": { "cart_id": { "type": "string" } }
      }
    }
  }
  ```
* **Response (`201 Created`):**
  ```json
  {
    "success": true,
    "status_code": 201,
    "message": "New schema version appended",
    "data": {
      "api_id": "018f23a4-8b91-7d12-b234-333333333333",
      "version": "v1.1.0",
      "sha256_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "is_revert": false,
      "reverted_from_version": null
    },
    "meta": { "timestamp": "2026-11-08T12:00:00Z" }
  }
  ```

---

### 3.5 High-Throughput AOF Telemetry Stream Module (`core/modules/telemetry`)

#### `POST /api/v1/telemetry/batch`
* **Responsibility:** Asynchronous compressed batch ingestion from client SDK local AOF / ring buffers. Writes directly to BadgerDB (`logs::{project_id}::{timestamp_nanos}::{log_id}`).
* **Auth Required:** Scoped Project API Key (`X-Project-Key` Header)
* **Request Body:**
  ```json
  {
    "project_id": "018f23a4-8b91-7d12-b234-111111111111",
    "batch_size": 2,
    "logs": [
      {
        "severity": "ERROR",
        "message": "Database connection pool exhausted",
        "stack_trace": "main.go:42 -> db.go:108",
        "timestamp_nanos": 1794052800000000000
      },
      {
        "severity": "WARN",
        "message": "High memory consumption detected",
        "stack_trace": "",
        "timestamp_nanos": 1794052800005000000
      }
    ]
  }
  ```
* **Response (`202 Accepted`):**
  ```json
  {
    "success": true,
    "status_code": 202,
    "message": "Telemetry AOF batch queued for LSM write",
    "data": {
      "ingested_count": 2
    },
    "meta": { "timestamp": "2026-11-08T12:00:00Z" }
  }
  ```

---
