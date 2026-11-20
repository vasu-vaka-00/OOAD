# Insurance Policy and Claims Processing System — Setup Guide

A complete, production-grade guide for cloning, configuring, migrating, running, and verifying the **Assurance: Insurance Policy and Claims Processing System** across backend services, PostgreSQL database, and React Native Expo mobile application.

---

## Table of Contents
1. [System Prerequisites](#1-system-prerequisites)
2. [Project Architecture & Directory Layout](#2-project-architecture--directory-layout)
3. [Backend Setup & Database Initialization](#3-backend-setup--database-initialization)
4. [Mobile Application Setup](#4-mobile-application-setup)
5. [Pre-Seeded Demo Accounts](#5-pre-seeded-demo-accounts)
6. [Monorepo & Verification Commands](#6-monorepo--verification-commands)
7. [Health-Check & Connectivity Verification](#7-health-check--connectivity-verification)
8. [Troubleshooting Guide](#8-troubleshooting-guide)

---

## 1. System Prerequisites

Before running the project, verify that your development workstation has the following tools installed:

| Tool | Recommended Version | Minimum Version | Verification Command |
|---|---|---|---|
| **Node.js** | `v20.x LTS` | `v18.0.0` | `node -v` |
| **npm** | `v10.x` | `v9.0.0` | `npm -v` |
| **PostgreSQL** | `v16.x` | `v14.0` | `psql --version` |
| **Git** | `v2.40+` | `v2.20` | `git --version` |
| **Expo CLI** | Integrated via `npx expo` | SDK 52 | `npx expo -v` |
| **EAS CLI** *(optional, for cloud builds)* | `v12.x+` | `v10.0` | `npx eas-cli -v` |
| **Java Development Kit (JDK)** *(for local Android builds)* | `OpenJDK 17` | `JDK 17` | `java -version` |
| **Android Studio & SDK** *(optional, for local emulator)* | Hedgehog / Iguana / Jellyfish | API Level 34 | `adb version` |

> [!NOTE]
> Physical mobile devices require the free **Expo Go** application installed from Google Play Store or Apple App Store, connected to the same Wi-Fi/LAN network as your development computer.

---

## 2. Project Architecture & Directory Layout

The repository is organized as a structured full-stack monorepo:

```
insurance-policy-claims-system/
├── backend/                         # Express.js + TypeScript + PostgreSQL REST API
│   ├── prisma/
│   │   ├── schema.prisma            # 15 Prisma relational models & 8 enums
│   │   ├── migrations/              # SQL schema migration history
│   │   └── seed.ts                  # Safe development seed script
│   ├── src/
│   │   ├── config/                  # Environment (env.ts) & Prisma Client (prisma.ts)
│   │   ├── controllers/             # Route handlers (auth, customer, policy, claims, admin, etc.)
│   │   ├── middlewares/             # JWT auth, RBAC authorization, upload, rate limiting
│   │   ├── routes/                  # Express router definitions
│   │   ├── services/                # ACID business logic & audit logging
│   │   ├── types/                   # Shared TypeScript interfaces & DTOs
│   │   ├── tests/                   # 7 automated test suites (325 tests)
│   │   ├── app.ts                   # Express application setup, CORS, health endpoint
│   │   └── index.ts                 # HTTP server bootstrap & graceful shutdown
│   ├── .env.example                 # Development environment variable template
│   ├── .env.production.example      # Production environment template with pooling & SSL
│   ├── package.json
│   └── tsconfig.json
├── mobile/                          # React Native + Expo + TypeScript Mobile Client
│   ├── assets/                      # Application icons, adaptive icons, and splash screens
│   ├── src/
│   │   ├── api/                     # Axios HTTP client, token storage & refresh interceptors
│   │   ├── components/              # Shared UI design system (buttons, inputs, cards, badges)
│   │   ├── context/                 # AuthContext (JWT tokens, role-based session state)
│   │   ├── navigation/              # React Navigation stacks (Auth, Customer, Agent, Admin)
│   │   ├── screens/                 # Mobile screen views organized by role
│   │   │   ├── auth/                # Login, Register, ForgotPassword, ResetPassword
│   │   │   ├── customer/            # Dashboard, Policies, Claims, Activities, Profile, etc.
│   │   │   ├── agent/               # Agent Dashboard, Assigned Caseload, Review screens
│   │   │   └── admin/               # Admin Dashboard, Claims Adjudication, Reports, Audit Logs
│   │   ├── types/                   # Frontend TypeScript interfaces matching backend models
│   │   └── utils/                   # Formatting, date utilities, currency helpers
│   ├── app.json                     # Expo configuration (App ID: com.insurance.claims)
│   ├── eas.json                     # EAS build profiles (preview APK & production AAB)
│   ├── package.json
│   └── tsconfig.json
├── docs/                            # System Engineering Documentation
│   ├── API_DOCUMENTATION.md         # Comprehensive REST API specifications & schemas
│   ├── DATABASE_DESIGN.md           # Entity Relationship Diagram & Data Dictionary
│   ├── DEPLOYMENT_GUIDE.md          # Docker, Cloud hosting & APK distribution guide
│   ├── SETUP_GUIDE.md               # This setup guide
│   └── TESTING_REPORT.md            # Comprehensive test execution & quality report
├── package.json                     # Monorepo root runner
└── README.md                        # Master project documentation
```

---

## 3. Backend Setup & Database Initialization

### Step 3.1: Navigate to Backend Directory
```bash
cd insurance-policy-claims-system/backend
```

### Step 3.2: Configure Environment Variables
Copy `.env.example` to create your local `.env`:
```bash
cp .env.example .env
```

Ensure `.env` contains the required parameters:
```env
# Server Configuration
PORT=5000
NODE_ENV=development

# PostgreSQL Connection String
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/insurance_db?schema=public"

# JWT Authentication
JWT_SECRET=super_secret_jwt_key_at_least_32_characters_long_for_security_2026
JWT_EXPIRES_IN=15m
REFRESH_TOKEN_EXPIRES_DAYS=7
BCRYPT_SALT_ROUNDS=10

# Security & Rate Limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100
AUTH_RATE_LIMIT_MAX=10

# CORS Whitelist (Optional in development)
CLIENT_ORIGIN="http://localhost:8081,http://localhost:19006"
```

### Step 3.3: Install Dependencies
```bash
npm install
```

### Step 3.4: Generate Prisma Client & Run Database Migrations
Make sure your PostgreSQL server is running. Create the database `insurance_db` if it does not already exist:
```bash
# In psql or PostgreSQL management client:
CREATE DATABASE insurance_db;
```

Run Prisma schema validation and generate the client:
```bash
npx prisma generate
```

Apply database migrations:
```bash
npx prisma migrate dev --name init
```

### Step 3.5: Populate Database with Safe Development Seed Data
The seed script populates 5 standard insurance categories, pre-configured insurance products (Auto, Health, Home, Life, Travel), and 3 role-tested accounts:
```bash
npm run prisma:seed
# or: npx prisma db seed
```

### Step 3.6: Start Backend in Development Mode
```bash
npm run dev
```
The console will display:
```
======================================================
  INSURANCE POLICY & CLAIMS BACKEND SERVICE STARTED   
======================================================
  Environment : development
  Port        : 5000
  Database    : Connected (PostgreSQL via Prisma ORM)
  Health Check: http://localhost:5000/health
======================================================
```

---

## 4. Mobile Application Setup

### Step 4.1: Navigate to Mobile Directory
Open a separate terminal window and navigate to the mobile folder:
```bash
cd insurance-policy-claims-system/mobile
```

### Step 4.2: Install Mobile Dependencies
```bash
npm install
```

### Step 4.3: Configure Mobile Environment Variables
Copy `.env.example` to create `.env`:
```bash
cp .env.example .env
```

Set the backend API URL based on your target device:

| Target Device | `EXPO_PUBLIC_API_URL` Value | Explanation |
|---|---|---|
| **Android Emulator** | `http://10.0.2.2:5000` | Android emulator loopback alias pointing to your host development PC |
| **Physical Device (Expo Go)** | `http://<YOUR_LAN_IP>:5000` (e.g. `http://192.168.1.150:5000`) | Connects over local Wi-Fi. Run `ipconfig` (Windows) or `ifconfig` (macOS/Linux) to find LAN IP |
| **iOS Simulator** | `http://localhost:5000` | iOS simulator shares the local host network stack |
| **Web Browser** | `http://localhost:5000` | Browser runs directly on localhost |

Example `.env`:
```env
EXPO_PUBLIC_API_URL=http://10.0.2.2:5000
EXPO_PUBLIC_API_TIMEOUT=15000
```

### Step 4.4: Launch Expo Development Server
```bash
npm run start
```
From the interactive Expo terminal menu:
- Press `a` to open on a running **Android Emulator**.
- Press `i` to open on an **iOS Simulator**.
- Press `w` to open in your default **Web Browser**.
- Scan the printed QR code with the **Expo Go** app on your physical mobile phone.

---

## 5. Pre-Seeded Demo Accounts

The database seed script sets up authentic credentials across all 3 user roles:

| Role | Email | Password | Intended Role Scope & Permissions |
|---|---|---|---|
| **Administrator** | `admin@assurance.com` | `AdminPass123!` | System dashboard, application underwriting adjudication, claim payouts, product creation, audit logs |
| **Insurance Agent** | `agent.smith@assurance.com` | `AgentPass123!` | Caseload review, assigned customer management, claim investigations, internal underwriting notes |
| **Customer** | `customer@assurance.com` | `CustomerPass123!` | Policy purchasing, claim filing, evidence document upload, status tracking, notifications |

---

## 6. Monorepo & Verification Commands

From the root directory (`insurance-policy-claims-system`), you can run all verification checks across both subprojects:

### Full TypeScript Compilation Check (0 Errors Required)
```bash
npm run typecheck
```
*Executes `tsc --noEmit` across backend and mobile simultaneously.*

### ESLint Code Quality Check (0 Errors, 0 Warnings Required)
```bash
npm run lint
```
*Validates code styling, strict typing, and syntax integrity.*

### Backend Automated Test Suites (325 Tests Across 7 Suites)
```bash
cd backend
npm test
```
*Runs all 7 test suites:*
1. `npm run test:auth` (36 tests)
2. `npm run test:customer` (23 tests)
3. `npm run test:insurance` (26 tests)
4. `npm run test:part7` (45 tests - Applications & Claims)
5. `npm run test:part8` (61 tests - Agent & Admin)
6. `npm run test:part9` (52 tests - Documents & Notifications)
7. `npm run test:part13` (82 tests - Monorepo E2E Integration)

---

## 7. Health-Check & Connectivity Verification

### Check Server & PostgreSQL Database Status
Run a `curl` or browser request to the backend health endpoint:
```bash
curl http://localhost:5000/health
```

Expected JSON Response (`200 OK`):
```json
{
  "status": "UP",
  "service": "insurance-claims-backend",
  "version": "1.0.0",
  "environment": "development",
  "timestamp": "2026-11-20T17:35:00.000Z",
  "uptimeSeconds": 128,
  "database": {
    "status": "CONNECTED",
    "latencyMs": 4
  },
  "system": {
    "nodeVersion": "v20.x",
    "memoryRssMb": 85,
    "heapUsedMb": 42,
    "heapTotalMb": 58
  }
}
```

---

## 8. Troubleshooting Guide

### Issue 1: `ECONNREFUSED 127.0.0.1:5432`
- **Cause**: PostgreSQL service is stopped or port `5432` is blocked.
- **Solution**:
  - Windows: Open `services.msc`, locate `postgresql-x64-XX`, and click **Start**.
  - macOS: Run `brew services start postgresql@16`.
  - Linux: Run `sudo systemctl start postgresql`.
  - Verify credentials in `backend/.env`.

### Issue 2: Mobile App Shows `Network Error` or Connection Timeout
- **Cause**: Mobile app cannot reach the backend API host.
- **Solution**:
  - If using **Android Emulator**: Set `EXPO_PUBLIC_API_URL=http://10.0.2.2:5000`. Do not use `localhost` because `localhost` refers to the Android device itself.
  - If using **Physical Phone via Expo Go**: Set `EXPO_PUBLIC_API_URL=http://<YOUR_LAN_IP>:5000`. Verify both phone and PC are on the same Wi-Fi network and that Windows Defender / macOS Firewall allows inbound traffic on port 5000.

### Issue 3: `Error: EPERM: operation not permitted` on Windows
- **Cause**: Windows file locking or telemetry permissions.
- **Solution**:
  - Set `$env:EXPO_NO_TELEMETRY="1"` in PowerShell before launching Expo.
  - Ensure development tools are run with appropriate user permissions.

### Issue 4: Metro Bundler Port 8081 In Use
- **Cause**: Another node or React Native process is occupying port 8081.
- **Solution**:
  - Run `npx expo start --port 8082` to start Metro on an alternate port.
