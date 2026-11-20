# REST API Documentation — Insurance Policy & Claims Processing System

A comprehensive API specification for the **Assurance: Insurance Policy and Claims Processing System** RESTful backend microservice.

---

## 1. General API Architecture

### 1.1 Base URLs
- **Direct Base URL**: `http://localhost:5000/api`
- **Versioned Base URL**: `http://localhost:5000/api/v1`
- **Health Check Endpoint**: `http://localhost:5000/health` or `http://localhost:5000/api/health`
- **Android Emulator Loopback**: `http://10.0.2.2:5000/api`

### 1.2 Authentication & Security Headers
All protected endpoints require an HTTP `Authorization` header containing a valid JWT Bearer token:
```http
Authorization: Bearer <jwt-access-token>
```

Additionally, all responses include enterprise-grade security headers configured via Helmet:
- `Strict-Transport-Security: max-age=15552000; includeSubDomains`
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: SAMEORIGIN`
- `Cross-Origin-Opener-Policy: same-origin`

### 1.3 Standard Response Formats

#### Success Envelope (`200 OK`, `201 Created`):
```json
{
  "success": true,
  "message": "Operation completed successfully.",
  "data": { ... },
  "meta": {
    "page": 1,
    "limit": 10,
    "total": 45,
    "totalPages": 5
  }
}
```

#### Error Envelope (`400`, `401`, `403`, `404`, `409`, `429`, `500`):
```json
{
  "success": false,
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "The requested claim was not found or is inaccessible.",
    "details": [
      {
        "field": "claimId",
        "issue": "Invalid UUID format"
      }
    ]
  }
}
```

### 1.4 Rate Limiting Standards
| Scope | Window | Limit | Target Endpoints |
|---|---|---|---|
| **Public Authentication** | 15 minutes | 10 requests | `/api/auth/login`, `/api/auth/register`, `/api/auth/forgot-password` |
| **Document Uploads** | 15 minutes | 50 requests | `/api/documents/upload` |
| **Standard User API** | 15 minutes | 150 requests | All authenticated customer/agent/admin endpoints |

---

## 2. System Health & Observability

### `GET /health` & `GET /api/health`
Monitors API server status, PostgreSQL database connection, latency, and memory consumption.
- **Access**: Public
- **Response (`200 OK` or `503 Service Unavailable`):**
  ```json
  {
    "status": "UP",
    "service": "insurance-claims-backend",
    "version": "1.0.0",
    "environment": "production",
    "timestamp": "2026-11-20T17:35:00.000Z",
    "uptimeSeconds": 3600,
    "database": {
      "status": "CONNECTED",
      "latencyMs": 3
    },
    "system": {
      "nodeVersion": "v20.x",
      "memoryRssMb": 92,
      "heapUsedMb": 45,
      "heapTotalMb": 60
    }
  }
  ```

---

## 3. Authentication & Session Management

### 3.1 `POST /api/auth/register`
Registers a new customer account.
- **Access**: Public (Rate-limited: 10/15min)
- **Role Enforcement**: Always forces role `CUSTOMER`. Any payload attempting to set `ADMIN` or `AGENT` is strictly rejected with `400 Bad Request`.
- **Request Body:**
  ```json
  {
    "email": "customer@example.com",
    "password": "SecurePassword123!",
    "firstName": "John",
    "lastName": "Doe",
    "phoneNumber": "+1-555-0199",
    "city": "Springfield",
    "state": "IL",
    "postalCode": "62704",
    "country": "USA",
    "annualIncome": 85000
  }
  ```
- **Response (`201 Created`):**
  ```json
  {
    "success": true,
    "message": "Account successfully registered.",
    "data": {
      "user": {
        "id": "c1f19d36-8bb0-4d7a-8fbb-58e137b12d5e",
        "email": "customer@example.com",
        "firstName": "John",
        "lastName": "Doe",
        "role": "CUSTOMER"
      },
      "tokens": {
        "accessToken": "eyJhbGciOiJIUzI1Ni...",
        "refreshToken": "48-byte-cryptographic-hex-string...",
        "tokenType": "Bearer",
        "expiresIn": "15m"
      }
    }
  }
  ```

### 3.2 `POST /api/auth/login`
Authenticates user credentials and issues access & refresh tokens.
- **Access**: Public (Rate-limited: 10/15min)
- **Request Body:**
  ```json
  {
    "email": "customer@example.com",
    "password": "SecurePassword123!"
  }
  ```
- **Response (`200 OK`):**
  ```json
  {
    "success": true,
    "message": "Login successful.",
    "data": {
      "user": {
        "id": "c1f19d36-8bb0-4d7a-8fbb-58e137b12d5e",
        "email": "customer@example.com",
        "firstName": "John",
        "lastName": "Doe",
        "role": "CUSTOMER"
      },
      "tokens": {
        "accessToken": "eyJhbGciOiJIUzI1Ni...",
        "refreshToken": "48-byte-cryptographic-hex-string...",
        "tokenType": "Bearer",
        "expiresIn": "15m"
      }
    }
  }
  ```

### 3.3 `POST /api/auth/refresh`
Performs secure cryptographic refresh token rotation.
- **Access**: Public (Requires valid unrevoked refresh token)
- **Request Body:**
  ```json
  {
    "refreshToken": "48-byte-cryptographic-hex-string..."
  }
  ```
- **Security**: The old refresh token is immediately marked `isRevoked = true`. If an already revoked token is submitted, the system triggers security mitigation and invalidates all session tokens for that user.
- **Response (`200 OK`):**
  ```json
  {
    "success": true,
    "message": "Token refreshed successfully.",
    "data": {
      "accessToken": "eyJhbGciOiJIUzI1Ni...",
      "refreshToken": "new-48-byte-cryptographic-hex-string...",
      "tokenType": "Bearer",
      "expiresIn": "15m"
    }
  }
  ```

### 3.4 `POST /api/auth/logout`
Revokes the caller's active refresh tokens.
- **Access**: Authenticated (`CUSTOMER`, `AGENT`, `ADMIN`)
- **Response (`200 OK`):**
  ```json
  {
    "success": true,
    "message": "Logged out successfully."
  }
  ```

### 3.5 `GET /api/auth/me`
Retrieves identity and demographic profile for the authenticated caller.
- **Access**: Authenticated
- **Response (`200 OK`):**
  ```json
  {
    "success": true,
    "data": {
      "user": {
        "id": "c1f19d36-8bb0-4d7a-8fbb-58e137b12d5e",
        "email": "customer@example.com",
        "role": "CUSTOMER",
        "customerProfile": {
          "id": "...",
          "city": "Springfield",
          "annualIncome": "85000.00"
        }
      }
    }
  }
  ```

---

## 4. Customer Module

### 4.1 `GET /api/customer/dashboard`
Returns live aggregates calculated directly from PostgreSQL (no hardcoded metrics).
- **Access**: `CUSTOMER`
- **Response (`200 OK`):**
  ```json
  {
    "success": true,
    "data": {
      "metrics": {
        "activePoliciesCount": 2,
        "pendingApplicationsCount": 1,
        "activeClaimsCount": 1,
        "approvedClaimsCount": 1,
        "unreadNotificationsCount": 3
      },
      "recentActivities": [
        {
          "id": "...",
          "title": "Claim Submitted",
          "timestamp": "2026-11-20T14:20:00.000Z"
        }
      ]
    }
  }
  ```

### 4.2 `GET /api/customer/profile` & `PUT /api/customer/profile`
Retrieve or update customer contact information and address.
- **Access**: `CUSTOMER`
- **Update Body (`PUT`):**
  ```json
  {
    "phoneNumber": "+1-555-0299",
    "streetAddress": "456 Oak Avenue",
    "city": "Chicago",
    "state": "IL",
    "postalCode": "60601",
    "annualIncome": 92000
  }
  ```

---

## 5. Insurance Product Management

### 5.1 `GET /api/insurance/categories`
Returns all 5 insurance categories with product counts and icon metadata.
- **Access**: Public / Authenticated

### 5.2 `GET /api/insurance`
Returns paginated catalog of active insurance products with filtering and search.
- **Query Parameters**:
  - `category`: `AUTO` | `HEALTH` | `HOME` | `LIFE` | `TRAVEL`
  - `search`: Keyword string matching title or description
  - `page`: Page index (default: 1)
  - `limit`: Page size (default: 10)
- **Response (`200 OK`):**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "e88d1d86-1d12-40be-bdcf-8e43486241ef",
        "code": "AUTO-SHIELD-01",
        "name": "Comprehensive Auto Shield",
        "category": "AUTO",
        "description": "Full collision, comprehensive, and liability coverage.",
        "minCoverage": "10000.00",
        "maxCoverage": "100000.00",
        "basePremium": "45.00",
        "standardDeductible": "500.00",
        "coverageDurationMonths": 12,
        "requiredDocuments": ["ID_PROOF", "VEHICLE_REGISTRATION", "DRIVING_LICENSE"],
        "isActive": true
      }
    ]
  }
  ```

### 5.3 `POST /api/insurance` (Admin Only)
Creates a new insurance product offering.
- **Access**: `ADMIN`

---

## 6. Policy Application Lifecycle

### 6.1 `POST /api/applications`
Customer submits an application for an insurance policy.
- **Access**: `CUSTOMER`
- **Request Body:**
  ```json
  {
    "productId": "e88d1d86-1d12-40be-bdcf-8e43486241ef",
    "coverageAmount": 50000,
    "paymentFrequency": "ANNUAL",
    "underwritingData": {
      "vehicleMake": "Toyota",
      "vehicleModel": "Camry",
      "vehicleYear": 2022,
      "estimatedMileage": 12000
    },
    "documentIds": ["c3f19d36-8bb0-4d7a-8fbb-58e137b12d5e"]
  }
  ```
- **Response (`201 Created`):**
  ```json
  {
    "success": true,
    "message": "Application submitted successfully.",
    "data": {
      "id": "a1b2c3d4-...",
      "applicationNumber": "APP-2026-98124",
      "status": "SUBMITTED",
      "premiumAmount": "540.00",
      "createdAt": "2026-11-20T15:00:00.000Z"
    }
  }
  ```

### 6.2 `GET /api/applications/my`
Returns all applications submitted by the logged-in customer.

### 6.3 `PATCH /api/applications/:id/review` (Agent / Admin)
Updates underwriting review status (`UNDER_REVIEW`, `RECOMMENDED_APPROVAL`, `RECOMMENDED_REJECTION`).

### 6.4 `POST /api/applications/:id/adjudicate` (Admin Only)
Formally approves or rejects an application.
- When `action: "APPROVE"`, the backend automatically generates a corresponding active `Policy` record with a unique policy number `POL-2026-XXXXX` and triggers customer notification.

---

## 7. Insurance Claims Processing

### 7.1 `POST /api/claims`
Policyholder files a claim against an active insurance policy.
- **Access**: `CUSTOMER`
- **Request Body:**
  ```json
  {
    "policyId": "b2c3d4e5-...",
    "incidentDate": "2026-11-18T10:30:00.000Z",
    "incidentDescription": "Vehicle collision at intersection. Moderate front-end bumper damage.",
    "incidentLocation": "Corner of 5th Ave and Main St",
    "claimedAmount": 4800,
    "evidenceDocumentIds": ["doc-uuid-1", "doc-uuid-2"]
  }
  ```
- **Response (`201 Created`):**
  ```json
  {
    "success": true,
    "message": "Claim filed successfully.",
    "data": {
      "id": "c9876543-...",
      "claimNumber": "CLM-2026-44321",
      "status": "SUBMITTED",
      "claimedAmount": "4800.00",
      "createdAt": "2026-11-20T16:00:00.000Z"
    }
  }
  ```

### 7.2 `POST /api/claims/:id/assign` (Admin Only)
Assigns a dedicated claims adjuster / agent to investigate the claim.

### 7.3 `PATCH /api/claims/:id/stage` (Agent / Adjuster)
Advances the claim through permitted investigative stages (`INVESTIGATION`, `DOCUMENTS_REQUIRED`).

### 7.4 `POST /api/claims/:id/adjudicate` (Admin Only)
Adjudicates the claim, calculates deductible, and sets approved net settlement amount.
- **Request Body:**
  ```json
  {
    "decision": "APPROVED",
    "approvedAmount": 4500,
    "deductibleAmount": 500,
    "adjudicationNotes": "Damage inspection confirmed. Deductible applied per schedule."
  }
  ```
- **Calculation**: `netSettlementAmount = approvedAmount - deductibleAmount` ($4,000.00).

### 7.5 `POST /api/claims/:id/settle` (Admin Only)
Executes financial disbursement payout and transitions claim status to `SETTLED`.

---

## 8. Secure Document Management

### 8.1 `POST /api/documents/upload`
Uploads evidence documents with magic byte signature verification.
- **Access**: Authenticated (`CUSTOMER`, `AGENT`, `ADMIN`)
- **Content-Type**: `multipart/form-data`
- **Security Guarantees**:
  - File size restricted to <= 10MB.
  - Allowed MIME types: `image/jpeg`, `image/png`, `application/pdf`.
  - Actual file header bytes inspected using magic byte validator (prevents file spoofing).
  - Stored with non-guessable cryptographic UUID filenames in private directory.
- **Response (`201 Created`):**
  ```json
  {
    "success": true,
    "data": {
      "document": {
        "id": "d1e2f3a4-...",
        "fileName": "d1e2f3a4-repair_bill.pdf",
        "originalName": "repair_bill.pdf",
        "mimeType": "application/pdf",
        "fileSize": 245120
      }
    }
  }
  ```

### 8.2 `GET /api/documents/:id/download`
Streams authorized document bytes. Non-owners and unauthorized actors receive `403 Forbidden`.

---

## 9. Agent Caseload Management

### `GET /api/agent/dashboard`
Returns caseload summary for the logged-in agent:
- Assigned applications count
- Assigned claims count
- Customer portfolio count

### `GET /api/agent/applications` & `GET /api/agent/claims`
Lists only records explicitly assigned to the requesting agent.

---

## 10. System Administration & Compliance

### `GET /api/admin/dashboard`
Comprehensive KPI overview:
- `totalCustomers`, `totalAgents`, `totalPolicies`
- `pendingApplications`, `totalClaims`, `approvedClaims`, `settledClaims`
- `totalPayoutVolume` (aggregate financial sum)

### `GET /api/admin/audit-logs`
Immutable compliance audit logs with pagination, actor ID, and action filters.

---

## 11. Real-Time In-App Notifications

### `GET /api/notifications`
Lists chronological notifications for the authenticated user.

### `GET /api/notifications/unread-count`
Returns unread count badge number.

### `PATCH /api/notifications/:id/read` & `PATCH /api/notifications/mark-all-read`
Updates read status of notifications.
