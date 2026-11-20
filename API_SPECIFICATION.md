# API Specification - Insurance Policy & Claims Processing System

## Base URLs
- Development Direct: `http://localhost:5000/api/auth`
- Development Versioned: `http://localhost:5000/api/v1/auth`
- LAN (Physical Device Testing): `http://<LAN_IP>:5000/api/auth`

## Authentication Header
Private endpoints require a Bearer token in the HTTP `Authorization` header:
```http
Authorization: Bearer <jwt-access-token>
```

---

## 1. Authentication & Identity Management Endpoints

### 1.1 `POST /api/auth/register`
Public registration for customer / policyholder accounts.
- **Access**: Public (Strict rate limiting: 10 requests per 15 min window)
- **Security**: Forces `role: CUSTOMER`. Rejects any attempt to escalate role to `ADMIN` or `AGENT`.
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
- **Response (201 Created):**
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
        "phoneNumber": "+1-555-0199",
        "role": "CUSTOMER",
        "isActive": true,
        "isEmailVerified": false,
        "createdAt": "2026-11-20T08:45:00.000Z",
        "updatedAt": "2026-11-20T08:45:00.000Z"
      },
      "tokens": {
        "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
        "refreshToken": "48-byte-cryptographic-hex-string...",
        "tokenType": "Bearer",
        "expiresIn": "15m"
      }
    }
  }
  ```
- **Errors**:
  - `400 Bad Request`: Validation failure (weak password, missing fields, invalid role injection).
  - `409 Conflict`: Email address is already registered.

---

### 1.2 `POST /api/auth/login`
Authenticates user credentials and issues an access token and refresh token bundle.
- **Access**: Public (Brute-force protected: 10 attempts per 15 min window)
- **Request Body:**
  ```json
  {
    "email": "customer@example.com",
    "password": "SecurePassword123!",
    "deviceInfo": "Expo Mobile / Pixel 8 (Android 14)"
  }
  ```
- **Response (200 OK):**
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
        "role": "CUSTOMER",
        "isActive": true
      },
      "tokens": {
        "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
        "refreshToken": "48-byte-cryptographic-hex-string...",
        "tokenType": "Bearer",
        "expiresIn": "15m"
      }
    }
  }
  ```
- **Errors**:
  - `401 Unauthorized`: Invalid email or password, or account inactive.

---

### 1.3 `POST /api/auth/refresh`
Performs secure **Refresh Token Rotation**. Issues a brand-new access token AND brand-new refresh token while invalidating the old one.
- **Access**: Public (Requires raw refresh token)
- **Security**: Includes **Compromise / Token Reuse Detection**. If a previously revoked refresh token is presented, the system terminates and invalidates **ALL** active refresh tokens belonging to the user family.
- **Request Body:**
  ```json
  {
    "refreshToken": "48-byte-cryptographic-hex-string...",
    "deviceInfo": "Expo Mobile / Pixel 8"
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Session refreshed successfully.",
    "data": {
      "user": { "id": "...", "email": "...", "role": "CUSTOMER" },
      "tokens": {
        "accessToken": "new-access-token...",
        "refreshToken": "new-rotated-refresh-token...",
        "tokenType": "Bearer",
        "expiresIn": "15m"
      }
    }
  }
  ```
- **Errors**:
  - `401 Unauthorized`: Token invalid, expired, or reuse alarm triggered.

---

### 1.4 `POST /api/auth/logout`
Revokes the presented refresh token session in the database.
- **Access**: Public / Authenticated
- **Request Body:**
  ```json
  {
    "refreshToken": "48-byte-cryptographic-hex-string..."
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Logged out successfully."
  }
  ```

---

### 1.5 `POST /api/auth/forgot-password`
Initiates password recovery. Generates a cryptographically secure 1-hour reset token.
- **Access**: Public
- **Security**: Never leaks whether an email exists (prevents account enumeration attacks).
- **Request Body:**
  ```json
  {
    "email": "customer@example.com"
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "If an account exists with that email address, password reset instructions have been sent."
  }
  ```

---

### 1.6 `POST /api/auth/reset-password`
Completes password recovery. Updates password hash and invalidates all existing user refresh token sessions on all devices.
- **Access**: Public
- **Request Body:**
  ```json
  {
    "token": "raw-reset-token-received-via-email-or-sms",
    "newPassword": "NewStrongPassword2026!"
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "message": "Password has been successfully updated. All active sessions have been invalidated. Please log in with your new password."
  }
  ```
- **Errors**:
  - `400 Bad Request`: Token expired, token invalid, or weak password.

---

### 1.7 `GET /api/auth/me`
Retrieves authenticated caller's profile and role data without exposing credentials.
- **Access**: Private (All authenticated roles: `CUSTOMER`, `AGENT`, `CLAIMS_ADJUSTER`, `ADMIN`)
- **Headers**: `Authorization: Bearer <access-token>`
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "user": {
        "id": "c1f19d36-8bb0-4d7a-8fbb-58e137b12d5e",
        "email": "customer@example.com",
        "firstName": "John",
        "lastName": "Doe",
        "role": "CUSTOMER",
        "isActive": true,
        "isEmailVerified": false,
        "createdAt": "2026-11-20T08:45:00.000Z",
        "updatedAt": "2026-11-20T08:45:00.000Z",
        "profile": {
          "id": "...",
          "city": "Springfield",
          "state": "IL"
        }
      }
    }
  }
  ```
- **Errors**:
  - `401 Unauthorized`: Token missing, expired, or invalid.

---

### 1.8 `POST /api/auth/create-agent`
Enables system administrators to create an insurance agent or adjuster profile with specific underwriting/claims capacity and category specializations.
- **Access**: Private (Strictly `ADMIN` only)
- **Headers**: `Authorization: Bearer <admin-access-token>`
- **Request Body:**
  ```json
  {
    "email": "agent.smith@insurance.internal",
    "password": "AgentSecurePassword2026!",
    "firstName": "Sarah",
    "lastName": "Smith",
    "phoneNumber": "+1-555-0101",
    "licenseNumber": "AG-IL-2026-9876",
    "agencyName": "Midwest Metro Assurance",
    "specialization": ["AUTO", "HOME"],
    "maxAssignedClaims": 40
  }
  ```
- **Response (201 Created):**
  ```json
  {
    "success": true,
    "message": "Agent account successfully created.",
    "data": {
      "user": {
        "id": "...",
        "email": "agent.smith@insurance.internal",
        "firstName": "Sarah",
        "lastName": "Smith",
        "role": "AGENT",
        "isActive": true
      }
    }
  }
  ```
- **Errors**:
  - `401 Unauthorized`: Missing or invalid token.
  - `403 Forbidden`: Caller does not hold `ADMIN` role.
  - `409 Conflict`: Email or license number already registered.

---

### 1.9 `POST /api/auth/create-admin`
Enables system administrators to onboard additional administrators.
- **Access**: Private (Strictly `ADMIN` only)
- **Headers**: `Authorization: Bearer <admin-access-token>`
- **Request Body:**
  ```json
  {
    "email": "admin2@insurance.internal",
    "password": "AdminSecurePassword2026!",
    "firstName": "David",
    "lastName": "Miller",
    "phoneNumber": "+1-555-0188"
  }
  ```
- **Response (201 Created):**
  ```json
  {
    "success": true,
    "message": "Admin account successfully created.",
    "data": {
      "user": {
        "id": "...",
        "email": "admin2@insurance.internal",
        "role": "ADMIN"
      }
    }
  }
  ```
- **Errors**:
  - `403 Forbidden`: Non-admin attempt.

---

## 2. Customer Module Endpoints

### 2.1 `GET /api/customer/dashboard`
Fetches real-time dashboard metrics calculated directly from PostgreSQL:
- Active policies count
- Pending applications count
- Active claims count
- Approved claims count
- Unread notifications count
- Recent real activities audit history
- Customer profile summary

- **Access**: Private (`CUSTOMER`, `ADMIN`)
- **Headers**: `Authorization: Bearer <access-token>`
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "customer": {
        "id": "c1f19d36-8bb0-4d7a-8fbb-58e137b12d5e",
        "email": "customer@example.com",
        "firstName": "John",
        "lastName": "Doe",
        "role": "CUSTOMER",
        "profile": {
          "city": "Springfield",
          "state": "IL",
          "occupation": "Software Engineer"
        }
      },
      "metrics": {
        "activePoliciesCount": 2,
        "pendingApplicationsCount": 1,
        "activeClaimsCount": 1,
        "approvedClaimsCount": 1,
        "unreadNotificationsCount": 3
      },
      "recentActivities": [
        {
          "id": "uuid",
          "action": "CLAIM_SUBMITTED",
          "entityName": "Claim",
          "entityId": "uuid",
          "createdAt": "2026-11-20T08:50:00.000Z"
        }
      ]
    }
  }
  ```

---

### 2.2 `GET /api/customer/profile`
Retrieves customer personal information, contact numbers, residential address, and underwriting demographics.
- **Access**: Private (`CUSTOMER`, `ADMIN`)
- **Headers**: `Authorization: Bearer <access-token>`

---

### 2.3 `PUT /api/customer/profile`
Updates customer details. Automatically logs an immutable `AuditLog` entry.
- **Access**: Private (`CUSTOMER`, `ADMIN`)
- **Headers**: `Authorization: Bearer <access-token>`
- **Request Body:**
  ```json
  {
    "firstName": "John",
    "lastName": "Doe",
    "phoneNumber": "+1-555-0199",
    "addressLine1": "742 Evergreen Terrace",
    "city": "Springfield",
    "state": "IL",
    "postalCode": "62704",
    "occupation": "Senior Architect",
    "annualIncome": 115000
  }
  ```

---

### 2.4 `GET /api/customer/activities`
Retrieves paginated audit and security operations executed on the customer's account.
- **Access**: Private (`CUSTOMER`, `ADMIN`)
- **Query Params**: `limit=20`, `page=1`

---

### 2.5 `GET /api/customer/notifications`
Retrieves user notifications and alerts.
- **Access**: Private (`CUSTOMER`, `ADMIN`)

---

### 2.6 `PATCH /api/customer/notifications/:id/read`
Marks a specific notification as read.
- **Access**: Private (`CUSTOMER`, `ADMIN`)

---

### 2.7 `PATCH /api/customer/notifications/read-all`
Marks all unread notifications as read.
- **Access**: Private (`CUSTOMER`, `ADMIN`)

---

## 3. Insurance Product Management Endpoints (PART 6)

### 3.1 `GET /api/insurance/categories`
Retrieves all 5 insurance categories (`HEALTH`, `LIFE`, `AUTO`, `TRAVEL`, `HOME`) with UI metadata, display names, taglines, icons, and real-time active product counts.
- **Access**: Public / Authenticated
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "HEALTH",
        "category": "HEALTH",
        "displayName": "Health Insurance",
        "tagline": "Comprehensive Family & Senior Medical Care",
        "description": "Inpatient hospitalization, preventative care, specialist consultation...",
        "icon": "medical-outline",
        "accentColor": "#0284c7",
        "badgeBg": "#e0f2fe",
        "productCount": 2
      }
    ]
  }
  ```

---

### 3.2 `GET /api/insurance`
Lists insurance products with full pagination, substring search (across title, code, description), category filtering, and coverage/premium range filters.
- **Access**: Public / Authenticated
- **Query Parameters**:
  - `category`: `HEALTH | AUTO | HOME | LIFE | TRAVEL` (optional)
  - `search`: Substring search query (optional)
  - `minCoverage`: Minimum coverage filter (optional)
  - `maxCoverage`: Maximum coverage filter (optional)
  - `minPremium`: Minimum base premium filter (optional)
  - `maxPremium`: Maximum base premium filter (optional)
  - `page`: Page number (default: `1`)
  - `limit`: Items per page (default: `10`, max `100`)
  - `sortBy`: `name | minCoverageAmount | maxCoverageAmount | basePremiumAmount | createdAt`
  - `sortOrder`: `asc | desc` (default: `asc`)
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "prod-uuid",
        "code": "AUTO-COMP-01",
        "name": "Comprehensive Motor Shield",
        "category": "AUTO",
        "categoryDisplayName": "Vehicle Insurance",
        "description": "Complete collision, comprehensive, roadside assistance...",
        "minCoverageAmount": 25000,
        "maxCoverageAmount": 150000,
        "basePremiumRate": 0.024,
        "basePremiumAmount": 720,
        "standardDeductible": 500,
        "termDurationMonths": 12,
        "termDurationYears": 1,
        "eligibility": "Licensed drivers aged 18+...",
        "requiredDocuments": ["Driver's License", "Vehicle Registration (RC)"],
        "termsAndConditions": "Covers collision, comprehensive perils...",
        "isActive": true
      }
    ],
    "pagination": {
      "total": 10,
      "page": 1,
      "limit": 10,
      "totalPages": 1,
      "hasNext": false,
      "hasPrev": false
    }
  }
  ```

---

### 3.3 `GET /api/insurance/:id`
Retrieves detailed insurance product schedule by ID or unique product code.
- **Access**: Public / Authenticated
- **Response (200 OK):** Detailed product record with eligibility, required documents, and terms.
- **Error (404 Not Found):** If product does not exist.

---

### 3.4 `POST /api/insurance`
Creates a new insurance product in the system catalog.
- **Access**: Admin Only (`ADMIN`)
- **Headers**: `Authorization: Bearer <admin-jwt>`
- **Request Body:**
  ```json
  {
    "code": "HLTH-SEN-02",
    "name": "Senior Citizen Gold Care",
    "category": "HEALTH",
    "description": "Specialized geriatric care, critical illness recovery...",
    "minCoverageAmount": 50000,
    "maxCoverageAmount": 500000,
    "basePremiumRate": 0.028,
    "basePremiumAmount": 1200,
    "standardDeductible": 750,
    "termDurationMonths": 12,
    "eligibility": "Individuals aged 60 to 80 years...",
    "requiredDocuments": ["Passport / Real ID", "Proof of Age", "Health Certificate"],
    "termsAndConditions": "Includes critical illness riders..."
  }
  ```
- **Response (201 Created):** Created product summary.

---

### 3.5 `PUT /api/insurance/:id`
Updates an existing insurance product.
- **Access**: Admin Only (`ADMIN`)
- **Headers**: `Authorization: Bearer <admin-jwt>`
- **Response (200 OK):** Updated product summary.

---

### 3.6 `PATCH /api/insurance/:id/deactivate`
Deactivates an insurance product, hiding it from public catalog views.
- **Access**: Admin Only (`ADMIN`)
- **Headers**: `Authorization: Bearer <admin-jwt>`
- **Response (200 OK):** Deactivated product with `isActive: false`.

---

### 3.7 `PATCH /api/insurance/:id/activate`
Re-activates an insurance product.
- **Access**: Admin Only (`ADMIN`)
- **Headers**: `Authorization: Bearer <admin-jwt>`
- **Response (200 OK):** Activated product with `isActive: true`.

---

## 4. Policy Applications & Insurance Claims Processing Endpoints

### 4.1 `POST /api/applications`
Submits a complete policy application with eligibility and document proofs.
- **Access**: Customer Only (`CUSTOMER`)
- **Headers**: `Authorization: Bearer <customer-jwt>`

### 4.2 `GET /api/applications/my`
Retrieves all policy applications submitted by the authenticated customer.
- **Access**: Customer Only (`CUSTOMER`)

### 4.3 `GET /api/applications/:id`
Retrieves detailed application data, status history, and underwriting notes.
- **Access**: Customer, Assigned Agent, or Admin

### 4.4 `POST /api/applications/:id/adjudicate`
Final underwriting approval or rejection by an authorized administrator. On approval, an active `Policy` record is atomically generated with a unique policy number.
- **Access**: Admin Only (`ADMIN`)

### 4.5 `POST /api/claims`
Files a claim against an active policy with incident facts, requested claim amount, and supporting documents.
- **Access**: Policyholder (`CUSTOMER`)

### 4.6 `GET /api/claims/my`
Retrieves all filed insurance claims for the policyholder.
- **Access**: Policyholder (`CUSTOMER`)

### 4.7 `GET /api/claims/:id`
Retrieves full claim dossier, attached evidence documents, and adjudication history timeline.
- **Access**: Claimant, Assigned Adjuster, or Admin

### 4.8 `POST /api/claims/:id/assign`
Assigns an adjuster to manage investigation and document collection.
- **Access**: Admin Only (`ADMIN`)

### 4.9 `POST /api/claims/:id/adjudicate`
Adjudicates claim with gross approved amount and applied deductible calculation.
- **Access**: Admin Only (`ADMIN`)

### 4.10 `POST /api/claims/:id/settle`
Disburses and closes approved claim payout with payment method and transaction reference.
- **Access**: Admin Only (`ADMIN`)

---

## 5. Insurance Agent Module Endpoints (`/api/agent`)

### 5.1 `GET /api/agent/dashboard`
Returns live assigned workload KPIs: assigned customers, pending applications, active claims, and recent activities.
- **Access**: Agent Only (`AGENT`, `CLAIMS_ADJUSTER`, `ADMIN`)
- **Headers**: `Authorization: Bearer <agent-jwt>`

### 5.2 `GET /api/agent/customers`
Returns list of customers strictly assigned to the requesting agent.
- **Access**: Agent Only (`AGENT`, `CLAIMS_ADJUSTER`, `ADMIN`)

### 5.3 `GET /api/agent/customers/:id`
Returns customer deep details if and only if assigned to the agent.
- **Access**: Agent Only (`AGENT`, `CLAIMS_ADJUSTER`, `ADMIN`)
- **Security**: Strict 403 Forbidden on unassigned customer records.

### 5.4 `GET /api/agent/applications`
Retrieves underwriting applications assigned to the agent.
- **Access**: Agent Only (`AGENT`, `ADMIN`)

### 5.5 `POST /api/agent/applications/:id/review`
Submits agent underwriting evaluation, internal notes, and recommendation (`APPROVE` or `REJECT`).
- **Access**: Agent Only (`AGENT`, `ADMIN`)

### 5.6 `GET /api/agent/claims`
Retrieves claims assigned to the adjuster.
- **Access**: Agent Only (`AGENT`, `CLAIMS_ADJUSTER`, `ADMIN`)

### 5.7 `POST /api/agent/claims/:id/review`
Advances claim review stage (`INVESTIGATION`, `DOCUMENTS_REQUIRED`, `UNDER_REVIEW`) with adjuster notes.
- **Access**: Agent Only (`AGENT`, `CLAIMS_ADJUSTER`, `ADMIN`)

### 5.8 `GET /api/agent/profile`
Retrieves agent license number, specialization categories, and agency details.
- **Access**: Agent Only (`AGENT`, `CLAIMS_ADJUSTER`, `ADMIN`)

### 5.9 `PUT /api/agent/profile`
Updates agency name, phone number, and specialization areas.
- **Access**: Agent Only (`AGENT`, `CLAIMS_ADJUSTER`, `ADMIN`)

---

## 6. Administrator Governance Endpoints (`/api/admin`)

### 6.1 `GET /api/admin/dashboard`
Returns 8 real-time platform metrics and financial aggregates from PostgreSQL:
- Total Customers
- Total Agents
- Total In-Force Policies
- Pending Applications
- Total Claims
- Approved Claims
- Rejected Claims
- Claims Awaiting Review
- Financial aggregates (total coverage value, collected premiums, paid settlements)
- **Access**: Admin Only (`ADMIN`)

### 6.2 `GET /api/admin/customers`
Paginated search and listing of all customers.
- **Access**: Admin Only (`ADMIN`)

### 6.3 `GET /api/admin/customers/:id`
Inspects complete customer profile, policies, claims, and audit history.
- **Access**: Admin Only (`ADMIN`)

### 6.4 `PATCH /api/admin/customers/:id/status`
Toggles customer account active/suspended state.
- **Access**: Admin Only (`ADMIN`)

### 6.5 `POST /api/admin/customers/:id/assign-agent`
Allocates an agent to a customer account.
- **Access**: Admin Only (`ADMIN`)

### 6.6 `GET /api/admin/agents`
Roster of all certified agents and adjusters with active caseloads.
- **Access**: Admin Only (`ADMIN`)

### 6.7 `POST /api/admin/agents`
Onboards a new insurance agent with bcrypt password hashing and license number.
- **Access**: Admin Only (`ADMIN`)

### 6.8 `PATCH /api/admin/agents/:id/status`
Toggles agent active/inactive certification status.
- **Access**: Admin Only (`ADMIN`)

### 6.9 `GET /api/admin/reports/analytics`
Actuarial performance metrics:
- Policy sales by category
- Portfolio loss ratio
- Claims distribution by status
- Average claim settlement size
- Application-to-policy conversion rate
- **Access**: Admin Only (`ADMIN`)

### 6.10 `GET /api/admin/audit-logs`
Tamper-evident system event log records with payload delta (`payloadBefore` vs `payloadAfter`).
- **Access**: Admin Only (`ADMIN`)

### 6.11 `GET /api/admin/settings` & `PUT /api/admin/settings`
Retrieves and updates operational parameters (automated underwriting, auto-assignment, claim thresholds, maintenance mode).
- **Access**: Admin Only (`ADMIN`)

---

## 7. Secure Document Management System (`/api/documents`)

Private, RBAC-governed, tamper-resistant document repository for policy underwriting and claims processing. Files are stored securely outside web roots, accessed solely through authenticated streaming endpoints with verified ownership.

### 7.1 Security Architecture
- **Storage Location**: Private directory (`storage/documents/`) inaccessible via direct URLs.
- **Filename Sanitization**: Uploaded files receive cryptographic UUID filenames (`doc_<uuid>.<ext>`) to thwart path traversal (`../`) and shell injection.
- **Magic Bytes Validation**: Validates underlying byte sequences (`%PDF-`, PNG header, JPEG SOI `0xFFD8FF`, WebP RIFF) to block malicious executables disguised with fake extensions.
- **Access Authorization**:
  - `CUSTOMER`: Exclusively view/download documents attached to their own claims or applications.
  - `AGENT` / `CLAIMS_ADJUSTER`: Access documents strictly attached to assigned claims or applications.
  - `ADMIN`: Global authorized access.
  - Unauthorized callers receive `403 Forbidden`.

### 7.2 `POST /api/documents/upload`
Uploads a document via `multipart/form-data` or JSON `fileBase64`.
- **Access**: Private (Authenticated Users)
- **Headers**: `Authorization: Bearer <token>`
- **Payload (multipart or JSON)**:
  - `file`: Binary file stream (max 10MB) OR `fileBase64`: Base64 string
  - `fileName`: Original file name (e.g. `medical_report.pdf`)
  - `mimeType`: `application/pdf`, `image/jpeg`, `image/png`, `image/webp`
  - `documentType`: `ID_PROOF | POLICY_CONTRACT | MEDICAL_REPORT | HOSPITAL_BILL | DAMAGE_PHOTO | REPAIR_ESTIMATE | INCIDENT_REPORT | POLICE_REPORT | INVOICE | CLAIM_SUPPORTING_DOCUMENT | OTHER`
  - `claimId`: Optional claim UUID to associate
  - `applicationId`: Optional application UUID to associate
- **Response (201 Created):**
  ```json
  {
    "success": true,
    "data": {
      "id": "7fa128be-...",
      "fileName": "medical_report.pdf",
      "storedFileName": "doc_7fa128be-...pdf",
      "mimeType": "application/pdf",
      "fileSizeBytes": 512000,
      "documentType": "MEDICAL_REPORT",
      "isVerified": false,
      "uploadedByUserId": "usr-123",
      "downloadUrl": "/api/documents/7fa128be-.../download",
      "viewUrl": "/api/documents/7fa128be-.../view",
      "createdAt": "2026-11-20T10:00:00Z"
    }
  }
  ```

---

### 7.3 `GET /api/documents/:id`
Retrieves document metadata after verifying ownership or assignment permissions.
- **Access**: Private (Owner, Assigned Agent, Admin)
- **Response (200 OK):** Document metadata object.

---

### 7.4 `GET /api/documents/:id/download` & `GET /api/documents/:id/view`
Streams authorized file binary with security headers (`Content-Disposition: attachment` or `inline`, `X-Content-Type-Options: nosniff`).
- **Access**: Private (Owner, Assigned Agent, Admin)
- **Security Check**: Returns `403 Forbidden` if caller lacks ownership or assignment permissions.

---

### 7.5 `POST /api/documents/associate`
Associates an existing uploaded document with a policy application or claim record.
- **Access**: Private (Owner or Admin)
- **Payload**:
  ```json
  {
    "documentId": "7fa128be-...",
    "claimId": "claim-uuid",
    "documentType": "HOSPITAL_BILL"
  }
  ```

---

### 7.6 `PATCH /api/documents/:id/verify`
Marks document authenticity and verification status.
- **Access**: Agent or Admin Only (`AGENT`, `CLAIMS_ADJUSTER`, `ADMIN`)
- **Payload**:
  ```json
  {
    "isVerified": true,
    "remarks": "Verified against hospital registry"
  }
  ```

---

### 7.7 `POST /api/claims/:id/request-documents`
Adjuster or Admin issues formal request for additional evidentiary documents, transitioning claim status to `DOCUMENTS_REQUIRED` and triggering push/in-app alert to customer.
- **Access**: Assigned Agent or Admin (`AGENT`, `CLAIMS_ADJUSTER`, `ADMIN`)
- **Payload**:
  ```json
  {
    "requestedDocuments": ["MEDICAL_REPORT", "HOSPITAL_BILL"],
    "instructions": "Please submit itemized hospital bills with diagnosis codes.",
    "deadlineDate": "2026-12-05T00:00:00Z"
  }
  ```

---

## 8. Notification & Event Alert System (`/api/notifications`)

Full-lifecycle event dispatch, in-app notification center, and sanitized Expo Push Notification delivery.

### 8.1 Event Trigger Architecture
Automatic notifications generated across real domain events:
1. `CUSTOMER_REGISTERED`: Welcome policyholder onboarding alert.
2. `APPLICATION_SUBMITTED`: Confirmation of underwriting submission.
3. `APPLICATION_APPROVED`: Policy underwriting approval notice.
4. `APPLICATION_REJECTED`: Application declination with rationale.
5. `ADDITIONAL_DOCUMENTS_REQUESTED`: Request for evidentiary documentation.
6. `DOCUMENTS_RESUBMITTED`: Receipt of resubmitted claimant documents.
7. `CLAIM_SUBMITTED`: Claim docket reference assignment.
8. `CLAIM_UNDER_REVIEW`: Review progression notice.
9. `CLAIM_APPROVED`: Claim adjudication and approved indemnity notice.
10. `CLAIM_REJECTED`: Claim declination notice with reason.
11. `CLAIM_SETTLED`: Payment disbursement and settlement completion.

### 8.2 Privacy & Sensitive Data Scrubbing
Push notification payloads strip PII and sensitive financial data (SSN, credit card, bank routing numbers) before dispatching to external push services.

---

### 8.3 `GET /api/notifications`
Retrieves paginated notifications for the authenticated user, filterable by unread status or event type.
- **Access**: Private (Authenticated User)
- **Query Parameters**:
  - `unreadOnly`: `true | false`
  - `page`: Page number (default: `1`)
  - `limit`: Items per page (default: `20`)
  - `eventType`: Filter by specific event type
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": [
      {
        "id": "notif-uuid",
        "userId": "usr-123",
        "title": "Claim Approved - CLM-2026-884920",
        "message": "Your claim for Comprehensive Motor Shield has been approved for $4,500.00.",
        "eventType": "CLAIM_APPROVED",
        "isRead": false,
        "entityType": "CLAIM",
        "entityId": "claim-uuid",
        "createdAt": "2026-11-20T10:15:00Z"
      }
    ],
    "pagination": {
      "total": 12,
      "page": 1,
      "limit": 20,
      "totalPages": 1
    }
  }
  ```

---

### 8.4 `GET /api/notifications/unread-count`
Returns the count of unread notifications for badge rendering.
- **Access**: Private (Authenticated User)
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "unreadCount": 3
    }
  }
  ```

---

### 8.5 `PATCH /api/notifications/:id/read`
Marks a specific notification as read.
- **Access**: Private (Owner)

---

### 8.6 `PATCH /api/notifications/mark-all-read`
Marks all notifications of the authenticated caller as read.
- **Access**: Private (Authenticated User)

---

### 8.7 `DELETE /api/notifications/:id`
Deletes a notification from user history.
- **Access**: Private (Owner or Admin)

---

## 9. Standard API Response Envelope & End-to-End Integration Verification

All backend endpoints strictly comply with a single, uniform API response contract:

### 9.1 Standard Success Envelope
```json
{
  "success": true,
  "message": "Optional human-readable confirmation message",
  "data": { ... },
  "pagination": {
    "total": 50,
    "page": 1,
    "limit": 20,
    "totalPages": 3,
    "hasNext": true,
    "hasPrev": false
  }
}
```

### 9.2 Standard Error Envelope
```json
{
  "success": false,
  "error": "Error description or rejection cause",
  "details": {
    "fieldName": ["Validation error message"]
  }
}
```

### 9.3 16-Phase End-to-End Lifecycle Verification (PART 13 Verified)
The following 16 phases have been verified through automated integration testing (`npm run test:part13` - 82 tests, 0 failures):
1. **Authentication & Authorization**: Registration, login, JWT rotation, identity inspection (`/api/auth/me`).
2. **Customer Profile & Demographics**: Demographic retrieval and persistent updates (`/api/customer/profile`).
3. **Customer Dashboard**: Real-time KPI metrics aggregation (`/api/customer/dashboard`).
4. **Insurance Catalog & Schedules**: 5 categories, product listing, range filters, and detailed schedules (`/api/insurance`).
5. **Policy Application Submission**: Atomic creation, validation, tracking number generation (`/api/applications`).
6. **Agent Caseload & Underwriting Review**: Work queue, internal notes, underwriting recommendations (`/api/agent/applications`).
7. **Admin Adjudication & Policy Issuance**: Official contract generation, policy issuance (`/api/applications/:id/adjudicate`).
8. **Secure Document Management**: Cryptographic UUID filenames, Magic Byte validation, strict RBAC isolation (`/api/documents/upload`).
9. **Insurance Claim Filing**: Multi-peril claim intake, reference number generation, coverage caps (`/api/claims`).
10. **Admin Adjuster Caseload Assignment**: Assignment workflow and audit logging (`/api/claims/:id/assign`).
11. **Adjuster Investigation & Document Requests**: Investigation transition, document requests (`/api/claims/:id/request-documents`).
12. **Customer Document Resubmission**: Intake of supplemental repair bills and invoices (`/api/claims/:id/resubmit-documents`).
13. **Admin Claim Adjudication & Settlement**: Deductible calculation, settlement disbursement (`/api/claims/:id/settle`).
14. **Notification Lifecycle**: Real-time event notifications, unread badges, mark-as-read (`/api/notifications`).
15. **Admin Governance & Audit Logs**: Platform-wide KPI counters, audit logging, system settings (`/api/admin`).
16. **Data Persistence**: Resilient state retention across server and client restarts.




